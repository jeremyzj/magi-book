# 热更新工作计划

热更新主要分为3部分工作， 
* hot-push-server(热更新服务器)，只要负责用户管理，RN包管理，管理更新统计，提供包下载业务
* hot-push-cli(热更新发布工具)，注册用户，注册APP以及发布RN包
* react-native-hot-push(客户端)，初始化热更服务，检查下载，下载替换本地包  

针对以上工作主要分为以下几期完成


## 一期 

完成基础的发布包更新功能，包版本控制，客户端获取对应的更新包，更新

### 1. hot-push-server(热更新服务器)
* 用户管理（注册，登录）
* APP管理（添加APP, 分配对应的deploykey）
* 包管理（提供上传RN包，检查是否有更新，包下载功能）

### 2. hot-push-cli(包发布工具)
* 添加用户
* 登录
* 添加APP获取delopmentKey
* 根据delopmentKey分发APP

### 3. react-native-hot-push(客户端包更新SDK)
* 根据delopmentKey检查是否包更新
* 下载包（建议wifi情况下载，后台静默下载）
* 用户重新打开APP，即可更新

## 二期

主要完成包更新统计，比如：牵牛2.4.1版本 发布RN包 V1.0，具体有多少设备下载，有多少设备激活

### 1. hot-push-server(热更新服务器)
* 设备管理
* 设备注册接口
* 包状态接口
* 基本的后台管理，用户查看统计报表(包下载、激活比例)

### 2. react-native-hot-push(客户端包更新SDK)
* 初始化注册
* 下载包时提交包状态信息（已下载）
* 更新成功以后提交包状态信息(已激活)

### 3. hot-push-cli(包发布工具)
* 查看发布包更新情况命令

## 三期 

协助管理：APP创建者做为owner的解决可以授权给其他协作developers
rollback: 对于整包更新APP rollback很重要

### 1. hot-push-server(热更新服务器)
* 增加协作developer管理
* 检查更新接口增加rollback信息

### 2. hot-push-cli(包发布工具)
* 新增协作者命令
* 增加rollback命令

### 3. react-native-hot-push(客户端包更新SDK)
* 获取rollback信息，回滚对应的包

### 四期：

灰度部署：对于APP这个很有意义，特别是新功能试用用户，以及针对特定机型和用户的BUG修复，达到一定的满意度，再全量部署。


### 1. hot-push-server(热更新服务器)
* 维度管理，增加特征标签
* 包管理中，增加标签属性

### 2. hot-push-cli(包发布工具)
* 发布命令中，增加标签参数



