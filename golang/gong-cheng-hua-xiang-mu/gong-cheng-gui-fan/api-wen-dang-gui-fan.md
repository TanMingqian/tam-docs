# API文档规范

**接口描述：**描述接口实现了什么功能。&#x20;

**请求方法**：接口的请求方法，格式为 HTTP 方法 请求路径，例如 POST /v1/users。在 通用说明中的请求方法部分，会说明接口的请求协议和请求地址。&#x20;

**输入参数：**接口的输入字段，它又分为 Header 参数、Query 参数、Body 参数、Path 参数。每个字段通过：参数名称、必选、类型 和 描述 4 个属性来描述。如果参数有限制或者默认值，可以在描述部分注明。&#x20;

**输出参数：**接口的返回字段，每个字段通过 参数名称、类型 和 描述 3 个属性来描述。&#x20;

**请求示例：**一个真实的 API 接口请求和返回示例。

### 示例

```
1.1 接口描述
创建用户。
1.2 请求方法
POST /v1/users
1.3 输入参数
Body 参数
参数名称	必选	类型	描述
metadata	是	ObjectMeta	REST 资源的功能属性
nickname	是	String	昵称
password	是	String	密码
email	是	String	邮箱地址
phone	否	String	电话号码
1.4 输出参数
参数名称	类型	描述
metadata	ObjectMeta	REST 资源的功能属性
nickname	String	昵称
password	String	密码
email	String	邮箱地址
phone	String	电话号码
1.5 请求示例
输入示例
 curl -XPOST -H'Content-Type: application/json'-H'Authorization: Bearer $Token'-d'{  "metadata": {    "name": "foo"  },  "nickname": "foo",  "password": "Foo@2020",  "email": "foo@foxmail.com",  "phone": "1812884xxxx"}'http://marmotedu.io:8080/v1/users
输出示例
 {  "metadata": {    "name": "foo",    "id": 31,    "createdAt": "2020-09-23T00:27:23.432346108+08:00",    "updatedAt": "2020-09-23T00:27:23.432346108+08:00"},  "nickname": "foo",  "password": "$2a$10$5M4m97yo4fZAHPwcRQdr1e0NaX7qMYKRIv0xePDtI8bk0ZGLN9X/6",  "email": "foo@foxmail.com",  "phone": "1812884xxxx"}

```

