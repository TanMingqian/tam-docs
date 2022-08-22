# go-web服务

![](<../../../.gitbook/assets/image (33).png>)

### 路由匹配

路由匹配其实就是根据(HTTP方法, 请求路径)匹配到处理这个请求的函数，最终由该函数处理这次请求

![](<../../../.gitbook/assets/image (1).png>)

### 一进程多服务

同时开启 HTTP 服务的 80 端口和 HTTPS 的 443 端口，这样我们就可以做到：对内的服务，访问 80 端口，简化服务访问复杂度；对外的服务，访问更为安全的 HTTPS 服务

把 ListenAndServe 函数放在 goroutine 中执行，并且调用 eg.Wait() 来阻塞程序进程，从而让两个 HTTP 服务在 goroutine 中持续监听端口，并提供服务。

```go
var eg errgroup.Group
insecureServer := &http.Server{...}
secureServer := &http.Server{...}

eg.Go(func() error {
  err := insecureServer.ListenAndServe()
  if err != nil && err != http.ErrServerClosed {
    log.Fatal(err)
  }
  return err
})
eg.Go(func() error {
  err := secureServer.ListenAndServeTLS("server.pem", "server.key")
  if err != nil && err != http.ErrServerClosed {
    log.Fatal(err)
  }
  return err
}

if err := eg.Wait(); err != nil {
  log.Fatal(err)
})
```

### 业务处理&#x20;

输入一些参数，校验通过后，进行业务逻辑处理，然后返回结果。所以 Web 服务还应该能够进行参数解析、参数校验、逻辑处理、返回结果。

#### 参数解析、参数校验、逻辑处理、返回结果

```go
func (u *productHandler) Create(c *gin.Context) {
  u.Lock()
  defer u.Unlock()

  // 1. 参数解析
  var product Product
  if err := c.ShouldBindJSON(&product); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
  }

  // 2. 参数校验
  if _, ok := u.products[product.Name]; ok {
    c.JSON(http.StatusBadRequest, gin.H{"error": fmt.Sprintf("product %s already exist", product.Name)})
    return
  }
  product.CreatedAt = time.Now()

  // 3. 逻辑处理
  u.products[product.Name] = product
  log.Printf("Register product %s success", product.Name)

  // 4. 返回结果
  c.JSON(http.StatusOK, product)
}

```

### Gin框架&#x20;

Gin 的官方文档 Gin是用 Go 语言编写的 Web 框架，功能完善，使用简单，性能很高。Gin 核心的路由功能是通过一个定制版的HttpRouter来实现的，具有很高的路由性能。

Gin 具有如下特性：&#x20;

轻量级，代码质量高，性能比较高；&#x20;

项目目前很活跃，并有很多可用的 Middleware；&#x20;

作为一个 Web 框架，功能齐全，使用起来简单。

### 路由分组&#x20;

Gin 通过 Group 函数实现了路由分组的功能。路由分组是一个非常常用的功能，可以将相同版本的路由分为一组，也可以将相同 RESTful 资源的路由分为一组。通过将路由分组，可以对相同分组的路由做统一处理。

```go
v1 := router.Group("/v1", gin.BasicAuth(gin.Accounts{"foo": "bar", "colin": "colin404"}))
{
    productv1 := v1.Group("/products")
    {
        // 路由匹配
        productv1.POST("", productHandler.Create)
        productv1.GET(":name", productHandler.Get)
    }

    orderv1 := v1.Group("/orders")
    {
        // 路由匹配
        orderv1.POST("", orderHandler.Create)
        orderv1.GET(":name", orderHandler.Get)
    }
}

v2 := router.Group("/v2", gin.BasicAuth(gin.Accounts{"foo": "bar", "colin": "colin404"}))
{
    productv2 := v2.Group("/products")
    {
        // 路由匹配
        productv2.POST("", productHandler.Create)
        productv2.GET(":name", productHandler.Get)
    }
}

```

### 中间件

![](<../../../.gitbook/assets/image (7).png>)

在 Gin 中，可以通过 gin.Engine 的 Use 方法来加载中间件。

中间件可以加载到不同的位置上，而且不同的位置作用范围也不同 Gin 框架本身支持了一些中间件。&#x20;

**gin.Logger()**：Logger 中间件会将日志写到 gin.DefaultWriter，gin.DefaultWriter 默认为 os.Stdout。&#x20;

**gin.Recovery()**：Recovery 中间件可以从任何 panic 恢复，并且写入一个 500 状态码。&#x20;

**gin.CustomRecovery(handle gin.RecoveryFunc)**：类似 Recovery 中间件，但是在恢复时还会调用传入的 handle 方法进行处理。&#x20;

**gin.BasicAuth()**：HTTP 请求基本认证（使用用户名和密码进行认证）

![](<../../../.gitbook/assets/image (3).png>)

### 认证、RequestID、跨域

认证、RequestID、跨域这三个高级功能，都可以通过 Gin 的中间件来实现

```go
router := gin.New()

// 认证
router.Use(gin.BasicAuth(gin.Accounts{"foo": "bar", "colin": "colin404"}))

// RequestID
router.Use(requestid.New(requestid.Config{
    Generator: func() string {
        return "test"
    },
}))

// 跨域
// CORS for https://foo.com and https://github.com origins, allowing:
// - PUT and PATCH methods
// - Origin header
// - Credentials share
// - Preflight requests cached for 12 hours
router.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://foo.com"},
    AllowMethods:     []string{"PUT", "PATCH"},
    AllowHeaders:     []string{"Origin"},
    ExposeHeaders:    []string{"Content-Length"},
    AllowCredentials: true,
    AllowOriginFunc: func(origin string) bool {
        return origin == "https://github.com"
    },
    MaxAge: 12 * time.Hour,
}))

```



