# 基础应用框架

### 应用框架

一个应用框架需要包含以下 3 个部分：&#x20;

**命令行参数解析**：主要用来解析命令行参数，这些命令行参数可以影响命令的运行效果。&#x20;

**配置文件解析**：一个大型应用，通常具有很多参数，为了便于管理和配置这些参数，通常会将这些参数放在一个配置文件中，供程序读取并解析。&#x20;

**应用的命令行框架**：应用最终是通过命令来启动的。这里有 3 个需求点，一是命令需要具备 Help 功能，这样才能告诉使用者如何去使用；二是命令需要能够解析命令行参数和配置文件；三是命令需要能够初始化业务代码，并最终启动业务进程。也就是说，我们的命令需要具备框架的能力，来纳管这 3 个部分。

命令行参数可以通过Pflag来解析，配置文件可以通过Viper来解析，应用的命令行框架则可以通过Cobra来实现。

{% embed url="https://github.com/spf13/pflag" %}

{% embed url="https://github.com/spf13/viper" %}

{% embed url="https://github.com/spf13/cobra" %}

Go 后端服务，基本上可以分为 API 服务和非 API 服务两类。&#x20;

**API 服务**：通过对外提供 HTTP/RPC 接口来完成指定的功能。比如订单服务，通过调用创建订单的 API 接口，来创建商品订单。&#x20;

**非 API 服务**：通过监听、定时运行等方式，而不是通过 API 调用来完成某些任务。比如数据处理服务，定时从 Redis 中获取数据，处理后存入后端存储中。再比如消息处理服务，监听消息队列（如 NSQ/Kafka/RabbitMQ），收到消息后进行处理。

### 启动流程

对于 API 服务和非 API 服务来说，它们的启动流程基本一致，都可以分为三步：&#x20;

应用框架的构建，这是最基础的一步。&#x20;

应用初始化。&#x20;

服务启动。

![](<../../../.gitbook/assets/image (23).png>)


