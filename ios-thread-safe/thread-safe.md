# 线程安全

## Apple 的框架
首先让我们来看看 Apple 的框架。一般来说除非特别声明，大多数的类默认都不是线程安全的。对于其中的一些类来说，这是很合理的，但是对于另外一些来说就很有趣了。

就算是在经验丰富的 iOS/Mac 开发者，也难免会犯从后台线程去访问 UIKit/AppKit 这种错误。比如因为图片的内容本身就是从后台的网络请求中获取的话，顺手就在后台线程中设置了 image 之类的属性，这样的错误其实是屡见不鲜的。Apple 的代码都经过了性能的优化，所以即使你从别的线程设置了属性的时候，也不会产生什么警告。

在设置图片这个例子中，症结其实是你的改变通常要过一会儿才能生效。但是如果有两个线程在同时对图片进行了设定，那么很可能因为当前的图片被释放两次，而导致应用崩溃。这种行为是和时机有关系的，所以很可能在开发阶段没有崩溃，但是你的用户使用时却不断 crash。

现在没有官方的用来寻找类似错误的工具，但我们确实有一些技巧来避免这个问题。[UIKit Main Thread Guard](https://gist.github.com/steipete/5664345) 是一段用来监视每一次对 setNeedsLayout 和 setNeedsDisplay 的调用代码，并检查它们是否是在主线程被调用的。因为这两个方法在 UIKit 的 setter （包括 image 属性）中广泛使用，所以它可以捕获到很多线程相关的错误。虽然这个小技巧并不包含任何私有 API， 但我们还是不建议将它是用在发布产品中，不过在开发过程中使用的话还是相当赞的。另外最新xcode已经提供了[UIKit, APPKit主线程渲染的检查](https://developer.apple.com/documentation/code_diagnostics/main_thread_checker)

Apple没有把 UIKit 设计为线程安全的类是有意为之的，将其打造为线程安全的话会使很多操作变慢。而事实上 UIKit 是和主线程绑定的，这一特点使得编写并发程序以及使用 UIKit 十分容易的，你唯一需要确保的就是对于 UIKit 的调用总是在主线程中来进行。


## 为什么 UIKit 不是线程安全的？
对于一个像 UIKit 这样的大型框架，确保它的线程安全将会带来巨大的工作量和成本。将 non-atomic 的属性变为 atomic 的属性只不过是需要做的变化里的微不足道的一小部分。通常来说，你需要同时改变若干个属性，才能看到它所带来的结果。为了解决这个问题，苹果可能不得不提供像 Core Data 中的 performBlock: 和 performBlockAndWait: 那样类似的方法来同步变更。另外你想想看，绝大多数对 UIKit 类的调用其实都是以配置为目的的，这使得将 UIKit 改为线程安全这件事情更显得毫无意义了。

然而即使是那些与配置共享的内部状态之类事情无关的调用，其实也不是线程安全的。如果你做过 iOS 3.2 或之前的黑暗年代的 app 开发的话，你肯定有过一边在后台准备图像时一边使用 NSString 的 drawInRect:withFont: 时的随机崩溃的经历。值得庆幸的事，在 iOS 4 中 苹果将大部分绘图的方法和诸如 UIColor 和 UIFont 这样的类改写为了后台线程可用。

但不幸的是 Apple 在线程安全方面的文档是极度匮乏的。他们推荐只访问主线程，并且甚至是绘图方法他们都没有明确地表示保证线程安全。因此在阅读文档的同时，去读读 iOS 版本更新说明会是一个很好的选择。

对于大多数情况来说，UIKit 类确实只应该用在应用的主线程中。这对于那些继承自 UIResponder 的类以及那些操作你的应用的用户界面的类来说，不管如何都是很正确的。


## 内存回收 (deallocation) 问题
另一个在后台使用 UIKit 对象的的危险之处在于“内存回收问题”。Apple 在技术笔记 TN2109 中概述了这个问题，并提供了多种解决方案。这个问题其实是要求 UI 对象应该在主线程中被回收，因为在它们的 dealloc 方法被调用回收的时候，可能会去改变 view 的结构关系，而如我们所知，这种操作应该放在主线程来进行。

因为调用者被其他线程持有是非常常见的（不管是由于 operation 还是 block 所导致的），这也是很容易犯错并且难以被修正的问题。在 AFNetworking 中也一直长久存在这样的 bug，但是由于其自身的隐蔽性而鲜为人知，也很难重现其所造成的崩溃。在异步的 block 或者操作中一致使用 __weak，并且不去直接访问局部变量会对避开这类问题有所帮助。


## Collection 类
Apple 有一个针对 iOS 和 Mac 的很好的总览性文档，为大多数基本的 foundation 类列举了其线程安全特性。总的来说，比如 NSArry 这样不可变类是线程安全的。然而它们的可变版本，比如 NSMutableArray 是线程不安全的。事实上，如果是在一个队列中串行地进行访问的话，在不同线程中使用它们也是没有问题的。要记住的是即使你申明了返回类型是不可变的，方法里还是有可能返回的其实是一个可变版本的 collection 类。一个好习惯是写类似于 return [array copy] 这样的代码来确保返回的对象事实上是不可变对象。

与和Java这样的语言不一样，Foundation 框架并不提供直接可用的 collection 类，这是有其道理的，因为大多数情况下，你想要的是在更高层级上的锁，以避免太多的加解锁操作。但缓存是一个值得注意的例外，iOS 4 中 Apple 添加的 NSCache 使用一个可变的字典来存储不可变数据，它不仅会对访问加锁，更甚至在低内存情况下会清空自己的内容。

也就是说，在你的应用中存在可变的且线程安全的字典是可以做到的。借助于 class cluster 的方式，我们也很容易写出这样的代码。

## 原子属性 (Atomic Properties)
你曾经好奇过 Apple 是怎么处理 atomic 的设置/读取属性的么？至今为止，你可能听说过自旋锁 (spinlocks)，信标 (semaphores)，锁 (locks)，@synchronized 等，Apple 用的是什么呢？因为 Objctive-C 的 runtime 是开源的，所以我们可以一探究竟。

一个非原子的 setter 看起来是这个样子的：
```Objc
- (void)setUserName:(NSString *)userName {
      if (userName != _userName) {
          [userName retain];
          [_userName release];
          _userName = userName;
      }
}
```
这是一个手动 retain/release 的版本，ARC 生成的代码和这个看起来也是类似的。当我们看这段代码时，显而易见要是 setUserName: 被并发调用的话会造成麻烦。我们可能会释放 _userName 两次，这回使内存错误，并且导致难以发现的 bug。

对于任何没有手动实现的属性，编译器都会生成一个 objc_setProperty_non_gc(id self, SEL _cmd, ptrdiff_t offset, id newValue, BOOL atomic, signed char shouldCopy) 的调用。在我们的例子中，这个调用的参数是这样的：
```Objc
objc_setProperty_non_gc(self, _cmd, 
  (ptrdiff_t)(&_userName) - (ptrdiff_t)(self), userName, NO, NO);`
```
ptrdiff_t 可能会吓到你，但是实际上这就是一个简单的指针算术，因为其实 Objective-C 的类仅仅只是 C 结构体而已。

objc_setProperty 调用的是如下方法：
```Objc
static inline void reallySetProperty(id self, SEL _cmd, id newValue, 
  ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy) 
{
    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:NULL];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:NULL];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spin_lock_t *slotlock = &PropertyLocks[GOODHASH(slot)];
        _spin_lock(slotlock);
        oldValue = *slot;
        *slot = newValue;        
        _spin_unlock(slotlock);
    }

    objc_release(oldValue);
}
```
除开方法名字很有趣以外，其实方法实际做的事情非常直接，它使用了在 PropertyLocks 中的 128 个自旋锁中的 1 个来给操作上锁。这是一种务实和快速的方式，最糟糕的情况下，如果遇到了哈希碰撞，那么 setter 需要等待另一个和它无关的 setter 完成之后再进行工作。

虽然这些方法没有定义在任何公开的头文件中，但我们还是可用手动调用他们。我不是说这是一个好的做法，但是知道这个还是蛮有趣的，而且如果你想要同时实现原子属性和自定义的 setter 的话，这个技巧就非常有用了。

// 手动声明运行时的方法
```Objc
extern void objc_setProperty(id self, SEL _cmd, ptrdiff_t offset, 
  id newValue, BOOL atomic, BOOL shouldCopy);
extern id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, 
  BOOL atomic);

#define PSTAtomicRetainedSet(dest, src) objc_setProperty(self, _cmd, 
  (ptrdiff_t)(&dest) - (ptrdiff_t)(self), src, YES, NO) 
#define PSTAtomicAutoreleasedGet(src) objc_getProperty(self, _cmd, 
  (ptrdiff_t)(&src) - (ptrdiff_t)(self), YES)
```
参考这个 gist 来获取包含处理结构体的完整的代码，但是我们其实并不推荐使用它。