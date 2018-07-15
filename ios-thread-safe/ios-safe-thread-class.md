# 线程安全类的设计

## 为何不用 @synchronized ？
你也许会想问为什么苹果不用 @synchronized(self) 这样一个已经存在的运行时特性来锁定属？？你可以看看这里的源码，就会发现其实发生了很多的事情。Apple 使用了最多三个加/解锁序列，还有一部分原因是他们也添加了异常开解(exception unwinding)机制。相比于更快的自旋锁方式，这种实现要慢得多。由于设置某个属性一般来说会相当快，因此自旋锁更适合用来完成这项工作。@synchonized(self) 更适合使用在你 需要确保在发生错误时代码不会死锁，而是抛出异常的时候。

## 你自己的类
单独使用原子属性并不会使你的类变成线程安全。它不能保护你应用的逻辑，只能保护你免于在 setter 中遭遇到竞态条件的困扰。看看下面的代码片段：

if (self.contents) {
    CFAttributedStringRef stringRef = CFAttributedStringCreate(NULL, 
      (__bridge CFStringRef)self.contents, NULL);
    // 渲染字符串
}
我之前在 PSPDFKit 中就犯了这个错误。时不时地应用就会因为 contents 属性在通过检查之后却又被设成了 nil 而导致 EXC_BAD_ACCESS 崩溃。捕获这个变量就可以简单修复这个问题；

NSString *contents = self.contents;
if (contents) {
    CFAttributedStringRef stringRef = CFAttributedStringCreate(NULL, 
      (__bridge CFStringRef)contents, NULL);
    // 渲染字符串
}
在这里这样就能解决问题，但是大多数情况下不会这么简单。想象一下我们还有一个 textColor 的属性，我们在一个线程中将两个属性都做了改变。我们的渲染线程有可能使用了新的内容，但是依旧保持了旧的颜色，于是我们得到了一组奇怪的组合。这其实也是为什么 Core Data 要将 model 对象都绑定在一个线程或者队列中的原因。

对于这个问题，其实没有万用解法。使用 不可变模型是一个可能的方案，但是它也有自己的问题。另一种途径是限制对存在在主线程或者某个特定队列中的既存对象的改变，而是先进行一次拷贝之后再在工作线程中使用。对于这个问题的更多对应方法，我推荐阅读 Jonathan Sterling 的关于 Objective-C 中轻量化不可变对象的文章。

一个简单的解决办法是使用 @synchronize。其他的方式都非常非常可能使你误入歧途，已经有太多聪明人在这种尝试上一次又一次地以失败告终。

## 可行的线程安全设计
在尝试写一些线程安全的东西之前，应该先想清楚是不是真的需要。确保你要做的事情不会是过早优化。如果要写的东西是一个类似配置类 (configuration class) 的话，去考虑线程安全这种事情就毫无意义了。更正确的做法是扔一个断言上去，以保证它被正确地使用：
```Objc
void PSPDFAssertIfNotMainThread(void) {
    NSAssert(NSThread.isMainThread, 
      @"Error: Method needs to be called on the main thread. %@", 
      [NSThread callStackSymbols]);
}
```
对于那些肯定应该线程安全的代码（一个好例子是负责缓存的类）来说，一个不错的设计是使用并发的 dispatch_queue 作为读/写锁，并且确保只锁着那些真的需要被锁住的部分，以此来最大化性能。一旦你使用多个队列来给不同的部分上锁的话，整件事情很快就会变得难以控制了。

于是你也可以重新组织你的代码，这样某些特定的锁就不再需要了。看看下面这段实现了一种多委托的代码（其实在大多数情况下，用 NSNotifications 会更好，但是其实也还是有多委托的实用例子）的
````Objc
// 头文件
@property (nonatomic, strong) NSMutableSet *delegates;

// init方法中
_delegateQueue = dispatch_queue_create("com.PSPDFKit.cacheDelegateQueue", 
  DISPATCH_QUEUE_CONCURRENT);

- (void)addDelegate:(id<PSPDFCacheDelegate>)delegate {
    dispatch_barrier_async(_delegateQueue, ^{
        [self.delegates addObject:delegate];
    });
}

- (void)removeAllDelegates {
    dispatch_barrier_async(_delegateQueue, ^{
        self.delegates removeAllObjects];
    });
}

- (void)callDelegateForX {
    dispatch_sync(_delegateQueue, ^{
        [self.delegates enumerateObjectsUsingBlock:^(id<PSPDFCacheDelegate> delegate, NSUInteger idx, BOOL *stop) {
            // 调用delegate
        }];
    });
}
```
除非 addDelegate: 或者 removeDelegate: 每秒要被调用上千次，否则我们可以使用一个相对简洁的实现方式：

```Objc
// 头文件
@property (atomic, copy) NSSet *delegates;

- (void)addDelegate:(id<PSPDFCacheDelegate>)delegate {
    @synchronized(self) {
        self.delegates = [self.delegates setByAddingObject:delegate];
    }
}

- (void)removeAllDelegates {
    @synchronized(self) {
        self.delegates = nil;
    }
}

- (void)callDelegateForX {
    [self.delegates enumerateObjectsUsingBlock:^(id<PSPDFCacheDelegate> delegate, NSUInteger idx, BOOL *stop) {
        // 调用delegate
    }];
}
```
就算这样，这个例子还是有点理想化，因为其他人可以把变更限制在主线程中。但是对于很多数据结构，可以在可变更操作的方法中创建不可变的拷贝，这样整体的代码逻辑上就不再需要处理过多的锁了。

GCD 的陷阱
对于大多数上锁的需求来说，GCD 就足够好了。它简单迅速，并且基于 block 的 API 使得粗心大意造成非平衡锁操作的概率下降了不少。然后，GCD 中还是有不少陷阱，我们在这里探索一下其中的一些。

将 GCD 当作递归锁使用
GCD 是一个对共享资源的访问进行串行化的队列。这个特性可以被当作锁来使用，但实际上它和 @synchronized 有很大区别。 GCD队列并非是可重入的，因为这将破坏队列的特性。很多有试图使用 dispatch_get_current_queue() 来绕开这个限制，但是这是一个糟糕的做法，Apple 在 iOS6 中将这个方法标记为废弃，自然也是有自己的理由。

```Objc
// This is a bad idea.
inline void pst_dispatch_sync_reentrant(dispatch_queue_t queue, 
  dispatch_block_t block) 
{
    dispatch_get_current_queue() == queue ? block() 
                                          : dispatch_sync(queue, block);
}
```
对当前的队列进行测试也许在简单情况下可以行得通，但是一旦你的代码变得复杂一些，并且你可能有多个队列在同时被锁住的情况下，这种方法很快就悲剧了。一旦这种情况发生，几乎可以肯定的是你会遇到死锁。当然，你可以使用 dispatch_get_specific()，这将截断整个队列结构，从而对某个特定的队列进行测试。要这么做的话，你还得为了在队列中附加标志队列的元数据，而去写自定义的队列构造函数。嘛，最好别这么做。其实在实用中，使用 NSRecursiveLock 会是一个更好的选择。

用 dispatch_async 修复时序问题
在使用 UIKit 的时候遇到了一些时序上的麻烦？很多时候，这样进行“修正”看来非常完美：

```Objc
dispatch_async(dispatch_get_main_queue(), ^{
    // Some UIKit call that had timing issues but works fine 
    // in the next runloop.
    [self updatePopoverSize];
});
```
千万别这么做！相信我，这种做法将会在之后你的 app 规模大一些的时候让你找不着北。这种代码非常难以调试，并且你很快就会陷入用更多的 dispatch 来修复所谓的莫名其妙的"时序问题"。审视你的代码，并且找到合适的地方来进行调用（比如在 viewWillAppear 里调用，而不是 viewDidLoad 之类的）才是解决这个问题的正确做法。我在自己的代码中也还留有一些这样的 hack，但是我为它们基本都做了正确的文档工作，并且对应的 issue 也被一一记录过。

记住这不是真正的 GCD 特性，而只是一个在 GCD 下很容易实现的常见反面模式。事实上你可以使用 performSelector:afterDelay: 方法来实现同样的操作，其中 delay 是在对应时间后的 runloop。

在性能关键的代码中混用 dispatch_sync 和 dispatch_async
这个问题我花了好久来研究。在 PSPDFKit 中有一个使用了 LRU（最久未使用）算法列表的缓存类来记录对图片的访问。当你在页面中滚动时，这个方法将被调用非常多次。最初的实现使用了 dispatch_sync 来进行实际有效的访问，使用 dispatch_async 来更新 LRU 列表的位置。这导致了帧数远低于原来的 60 帧的目标。

当你的 app 中的其他运行的代码阻挡了 GCD 线程的时候，dispatch manager 需要花时间去寻找能够执行 dispatch_async 代码的线程，这有时候会花费一点时间。在找到合适的执行线程之前，你的同步调用就会被 block 住了。其实在这个例子中，异步情况的执行顺序并不是很重要，但没有能将这件事情告诉 GCD 的好办法。读/写锁这里并不能起到什么作用，因为在异步操作中基本上一定会需要进行顺序写入，而在此过程中读操作将被阻塞住。如果误用了 dispatch_async 代价将会是非常惨重的。在将它用作锁的时候，一定要非常小心。

使用 dispatch_async 来派发内存敏感的操作
我们已经谈论了很多关于 NSOperations 的话题了，一般情况下，使用这个更高层级的 API 会是一个好主意。当你要处理一段内存敏感的操作的代码块时，这个优势尤为突出、

在 PSPDFKit 的老版本中，我用了 GCD 队列来将已缓存的 JPG 图片写到磁盘中。当 retina 的 iPad 问世之后，这个操作出现了问题。ß因为分辨率翻倍了，相比渲染这张图片，将它编码花费的时间要长得多。所以，操作堆积在了队列中，当系统繁忙时，甚至有可能因为内存耗尽而崩溃。

我们没有办法追踪有多少个操作在队列中等待运行（除非你手动添加了追踪这个的代码），我们也没有现成的方法来在接收到低内存通告的时候来取消操作、这时候，切换到 NSOperations 可以使代码变得容易调试得多，并且允许我们在不添加手动管理的代码的情况下，做到对操作的追踪和取消。

当然也有一些不好的地方，比如你不能在你的 NSOperationQueue 中设置目标队列（就像 DISPATCH_QUEUE_PRIORITY_BACKGROUND 之于 缓速 I/O 那样）。但这只是为了可调试性的一点小代价，而事实上这也帮助你避免遇到优先级反转的问题。我甚至不推荐直接使用已经包装好的 NSBlockOperation 的 API，而是建议使用一个 NSOperation 的真正的子类，包括实现其 description。诚然，这样做工作量会大一些，但是能输出所有运行中/准备运行的操作是及其有用的。