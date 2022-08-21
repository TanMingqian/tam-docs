# Swagger

Swagger 是一套围绕 OpenAPI 规范构建的开源工具，可以设计、构建、编写和使用 REST API&#x20;

**Swagger 编辑器**：基于浏览器的编辑器，可以在其中编写 OpenAPI 规范，并实时预览 API 文档。https://editor.swagger.io 就是一个 Swagger 编辑器，你可以尝试在其中编辑和预览 API 文档。&#x20;

**Swagger UI**：将 OpenAPI 规范呈现为交互式 API 文档，并可以在浏览器中尝试 API 调用。 Swagger Codegen：根据 OpenAPI 规范，生成服务器存根和客户端代码库，目前已涵盖了 40 多种语言。

## Swagger 和 OpenAPI 的区别

OpenAPI 是一个 API 规范，它的前身叫 Swagger 规范，通过定义一种用来描述 API 格式或 API 定义的语言，来规范 RESTful 服务开发过程

```
• OpenAPI 规范规定了一个 API 必须包含的基本信息，这些信息包括：
• 对 API 的描述，介绍 API 可以实现的功能。
• 每个 API 上可用的路径（/users）和操作（GET /users，POST /users）。
• 每个 API 的输入 / 返回的参数。
• 验证方法。
• 联系信息、许可证、使用条款和其他信息。
```

OpenAPI 是一个 API 规范，Swagger 则是实现规范的工具

## go-swagger

通过工具生成 Swagger 文档&#x20;

```shell
go get -u github.com/go-swagger/go-swagger/cmd/swagger 
go install github.com/go-swagger/go-swagger/cmd/swagger
```

![](<../../../.gitbook/assets/image (5).png>)

go-swagger 通过解析源码中的注释来生成 Swagger 文档

![](<../../../.gitbook/assets/image (14).png>)





