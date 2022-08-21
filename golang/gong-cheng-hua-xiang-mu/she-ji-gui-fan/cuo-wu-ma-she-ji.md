# 错误码设计

### 错误码&#x20;

有业务 Code 码标识&#x20;

对外对内分别展示不同的错误信息

### 常见的错误码设计方式

#### 第一种方式

不论请求成功或失败，始终返回200 http status code

```json
{ "error": { "message": "Syntax error "Field picture specified more than once. This is only possible before version 2.1" at character 23: id,name,picture,picture", "type": "OAuthException", "code": 2500, "fbtrace_id": "xxxxxxxxxxx" } }
```

### 第二种方式

返回http 404 Not Found错误码，并在 Body 中返回简单的错误信息

```json
HTTP/1.1 400 Bad Request x-connection-hash: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx set-cookie: guest_id=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx Date: Thu, 01 Jun 2017 03:04:23 GMT Content-Length: 62 x-response-time: 5 strict-transport-security: max-age=631138519 Connection: keep-alive Content-Type: application/json; charset=utf-8 Server: tsa_b
{"errors":[{"code":215,"message":"Bad Authentication data."}]}
```

#### 第三种方式

返回http 404 Not Found错误码，并在 Body 中返回详细的错误信息

```json
HTTP/1.1 400 Date: Thu, 01 Jun 2017 03:40:55 GMT Content-Length: 276 Connection: keep-alive Content-Type: application/json; charset=utf-8 Server: Microsoft-IIS/10.0 X-Content-Type-Options: nosniff
{"SearchResponse":{"Version":"2.2","Query":{"SearchTerms":"api error codes"},"Errors":[{"Code":1001,"Message":"Required parameter is missing.","Parameter":"SearchRequest.AppId","HelpUrl":"http\u003a\u002f\u002fmsdn.microsoft.com\u002fen-us\u002flibrary\u002fdd251042.aspx"}]}}
```

### 错误码设计建议

<pre><code>• 有区别于http status code的业务码，业务码需要有一定规则，可以通过业务码判断出是哪类错误。
• 请求出错时，可以通过http status code直接感知到请求出错。
• 需要在请求出错时，返回详细的信息，通常包括 3 类信息：业务 Code 码、错误信息和参考文档（可选）。
• 返回的错误信息，需要是可以直接展示给用户的安全信息，也就是说不能包含敏感信息；同时也要有内部更详细的错误信息，方便 debug。
• 返回的数据格式应该是固定的、规范的。
<strong>• 错误信息要保持简洁，并且提供有用的信息。</strong></code></pre>

### 业务 Code 码设计

<pre><code><strong>Code 码设计规范：纯数字表示，不同部位代表不同的服务，不同的模块。
</strong>错误代码说明：100101
10: 服务。
01: 某个服务下的某个模块。
01: 模块下的错误码序号，每个模块可以注册 100 个错误。通过100101可以知道这个错误是服务 A，数据库模块下的记录没有找到错误</code></pre>

### 设置 HTTP Status Code

<pre><code>• 1XX - （指示信息）表示请求已接收，继续处理。
• 2XX - （请求成功）表示成功处理了请求的状态代码。
• 3XX - （请求被重定向）表示要完成请求，需要进一步操作。通常，这些状态代码用来重定向。
• 4XX - （请求错误）这些状态代码表示请求可能出错，妨碍了服务器的处理，通常是客户端出错，需要客户端做进一步的处理。
<strong>• 5XX - （服务器错误）这些状态代码表示服务器在尝试处理请求时发生内部错误。这些错误可能是服务器本身的错误，而不是客户端的问题。</strong></code></pre>

```json
{
  "code": 100101,
  "message": "Database error",
  "reference": "https://github.com/marmotedu/iam/tree/master/docs/guide/zh-CN/faq/iam-apiserver"
}
```







