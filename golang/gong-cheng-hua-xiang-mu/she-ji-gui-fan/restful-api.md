# RESTful API

REST 代表的是**表现层状态转移（REpresentational State Transfer）**，由 Roy Fielding 在他的论文《Architectural Styles and the Design of Network-based Software Architectures》里提出。

REST 本身并没有创造新的技术、组件或服务，它只是一种**软件架构风格**，是一组架构约束条件和原则，而不是技术框架。

REST 有一系列规范，满足这些规范的 API 均可称为 RESTful API REST 规范把所有内容都视为资源，也就是说网络上一切皆资源。

REST 架构对资源的操作包括**获取、创建、修改和删除**，这些操作正好对应 HTTP 协议提供的 **GET、POST、PUT** 和 **DELETE** 方法

![](<../../../.gitbook/assets/image (28).png>)

## 核心特点：&#x20;

* 以资源 (resource) 为中心，所有的东西都抽象成资源，所有的行为都应该是在资源上的 CRUD 操作
* 资源对应着面向对象范式里的对象，面向对象范式以对象为中心。
* 资源使用 URI 标识，每个资源实例都有一个唯一的 URI 标识。例如，如果我们有一个用户，用户名是 admin，那么它的 URI 标识就可以是 /users/admin。
* 资源是有状态的，使用 JSON/XML 等在 HTTP Body 里表征资源的状态。
* 客户端通过四个 HTTP 动词，对服务器端资源进行操作，实现“表现层状态转化”
* 无状态，这里的无状态是指每个 RESTful API 请求都包含了所有足够完成本次操作的信息，服务器端无须保持 session。无状态对于服务端的弹性扩容是很重要的。

## 设计原则

### 1、URI设计

```
• 资源名使用名词而不是动词，并且用名词复数表示。资源分为 Collection 和 Member 两种。Collection：一堆资源的集合。例如我们系统里有很多用户（User）, 这些用户的集合就是 Collection。Collection 的 URI 标识应该是 域名/资源名复数, 例如https:// iam.api.marmotedu.com/users。Member：单个特定资源。例如系统中特定名字的用户，就是 Collection 里的一个 Member。Member 的 URI 标识应该是 域名/资源名复数/资源名称, 例如https:// iam.api.marmotedu/users/admin。
• URI 结尾不应包含/。
• URI 中不能出现下划线 _，必须用中杠线 -代替（有些人推荐用 _，有些人推荐用 -，统一使用一种格式即可，我比较推荐用 -）。
• URI 路径用小写，不要用大写。
• 避免层级过深的 URI。超过 2 层的资源嵌套会很乱，建议将其他资源转化为?参数
• 将一个操作变成资源的一个属性，比如想在系统中暂时禁用某个用户，可以这么设计 URI：/users/zhangsan?active=false。
• 将操作当作是一个资源的嵌套资源
• 如果以上都不能解决问题，有时可以打破这类规范。比如登录操作，登录不属于任何一个资源，URI 可以设计为：/login。
```

### 2、REST 资源操作映射为 HTTP 方法



![](<../../../.gitbook/assets/image (25).png>)

### 3、统一的返回格式

一个系统的 RESTful API 会向外界开放多个资源的接口，每个接口的返回格式要保持一致。另外，每个接口都会返回成功和失败两种消息，这两种消息的格式也要保持一致。

### 4、API版本管理

引入 API 版本机制，当不能向下兼容时，就引入一个新的版本，老的版本则保留原样。这样既能保证服务的可用性和安全性，同时也能满足新需求。

API 版本有不同的标识方法，在 RESTful API 开发中，通常将版本标识放在如下 3 个位置： URL 中，比如/v1/users。 HTTP Header 中，比如Accept: vnd.example-com.foo+json; version=1.0。 Form 参数中，比如/users?version=v1。

### 5、API命名

驼峰命名法 (serverAddress)、蛇形命名法 (server\_address) 和脊柱命名法 (server-address)

### 6、统一分页/过滤/排序/搜索功能

```
• 分页：在列出一个 Collection 下所有的 Member 时，应该提供分页功能，例如/users?offset=0&limit=20（limit，指定返回记录的数量；offset，指定返回记录的开始位置）。引入分页功能可以减少 API 响应的延时，同时可以避免返回太多条目，导致服务器 / 客户端响应特别慢，甚至导致服务器 / 客户端 crash 的情况。
• 过滤：如果用户不需要一个资源的全部状态属性，可以在 URI 参数里指定返回哪些属性，例如/users?fields=email,username,address。
• 排序：用户很多时候会根据创建时间或者其他因素，列出一个 Collection 中前 100 个 Member，这时可以在 URI 参数中指明排序参数，例如/users?sort=age,desc。
• 搜索：当一个资源的 Member 太多时，用户可能想通过搜索，快速找到所需要的 Member，或着想搜下有没有名字为 xxx 的某类资源，这时候就需要提供搜索功能。搜索建议按模糊匹配来搜索。
```

### 7、域名

https://marmotedu.com/api&#x20;

这种方式适合 API 将来不会有进一步扩展的情况，比如刚开始 marmotedu.com 域名下只有一套 API 系统，未来也只有这一套 API 系统。&#x20;

https://iam.api.marmotedu.com

如果 marmotedu.com 域名下未来会新增另一个系统 API，这时候最好的方式是每个系统的 API 拥有专有的 API 域名，比如：storage.api.marmotedu.com，network.api.marmotedu.com。腾讯云的域名就是采用这种方式。





