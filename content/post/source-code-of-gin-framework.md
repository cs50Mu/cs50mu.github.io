+++
title = "Gin源码分析"
date = "2019-10-27"
slug = "2019/10/27/source-code-of-gin-framework"
Categories = []
+++

github仓库：[https://github.com/gin-gonic/gin](https://github.com/gin-gonic/gin)

### what

Gin is a web framework written in Go (Golang). It features a martini-like API with much better performance, up to 40 times faster thanks to httprouter

### how

从一个例子开始：

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080
}
```
先看看Default()是如何实现的

```go
  // Default returns an Engine instance with the Logger and Recovery middleware already attached.
  func Default() *Engine {
      debugPrintWARNINGDefault()
      engine := New()
      // 默认创建的engine自带了logger和recovery中间件
      // 注意这里设置的是全局的中间件，每一个request都会被
      // 在这里设置的中间件处理
      engine.Use(Logger(), Recovery())
      return engine
  }
```
然后我们来看看创建的engine长什么样

```go
  // Engine is the framework's instance, it contains the muxer, middleware and configuration settings.
  // Create an instance of Engine, by using New() or Default()
  type Engine struct {
      RouterGroup
      // 篇幅原因，以下字段省略
      //...
   } 
   
  // Use attaches a global middleware to the router. ie. the middleware attached though Use() will be
  // included in the handlers chain for every single request. Even 404, 405, static files...
  // For example, this is the right place for a logger or error management middleware.
  func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
      // 底层调用的是RouterGroup的方法
      engine.RouterGroup.Use(middleware...)
      // 404时的handler
      engine.rebuild404Handlers()
      // 405时的handler
      engine.rebuild405Handlers()
      return engine
  }  
  
 // Use adds middleware to the group, see example code in GitHub.
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
    group.Handlers = append(group.Handlers, middleware...)
    return group.returnObj()
}
  
  func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
    finalSize := len(group.Handlers) + len(handlers)
    if finalSize >= int(abortIndex) {
        panic("too many handlers")
    }
    mergedHandlers := make(HandlersChain, finalSize)
    copy(mergedHandlers, group.Handlers)
    copy(mergedHandlers[len(group.Handlers):], handlers)
    return mergedHandlers
}
      
```
可以看到它内嵌了一个结构体`RouterGroup`，那么它是干嘛的呢？

```go
// RouterGroup is used internally to configure router, a RouterGroup is associated with
// a prefix and an array of handlers (middleware).
type RouterGroup struct {
    Handlers HandlersChain
    basePath string
    engine   *Engine
    root     bool
}
```
从注释上看，这就是负责路由映射的了。上面示例代码中的`r.GET`实际调用的是`RouterGroup`的方法(thanks to struct embedding)

```go  
  // GET is a shortcut for router.Handle("GET", path, handle).
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle("GET", relativePath, handlers)
}
  
  func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    absolutePath := group.calculateAbsolutePath(relativePath)
    // 如果在group上定义了共用的handler，比如logger和recovery此处会和传入的handlers一同返回
    handlers = group.combineHandlers(handlers)
    // 这里应该是建立请求路径和handler的映射关系
    // 涉及到Redix tree算法，感兴趣的可以具体看看
    group.engine.addRoute(httpMethod, absolutePath, handlers)
    return group.returnObj()
}
```

再看一个复杂一点的例子：

```go
func main() {
	// Creates a router without any middleware by default
	r := gin.New()

	// Global middleware
	// Logger middleware will write the logs to gin.DefaultWriter even if you set with GIN_MODE=release.
	// By default gin.DefaultWriter = os.Stdout
	r.Use(gin.Logger())

	// Recovery middleware recovers from any panics and writes a 500 if there was one.
	r.Use(gin.Recovery())

	// Per route middleware, you can add as many as you desire.
	r.GET("/benchmark", MyBenchLogger(), benchEndpoint)

	// Authorization group
	// authorized := r.Group("/", AuthRequired())
	// exactly the same as:
	authorized := r.Group("/")
	// per group middleware! in this case we use the custom created
	// AuthRequired() middleware just in the "authorized" group.
	authorized.Use(AuthRequired())
	{
		authorized.POST("/login", loginEndpoint)
		authorized.POST("/submit", submitEndpoint)
		authorized.POST("/read", readEndpoint)

		// nested group
		testing := authorized.Group("testing")
		testing.GET("/analytics", analyticsEndpoint)
	}

	// Listen and serve on 0.0.0.0:8080
	r.Run(":8080")
}
```

从上面的例子中，我们知道还可以定义组的概念（对应url中有统一前缀的情况），而且还可以针对这个组来绑定统一的middleware，那么是怎么实现的呢？

```go
// Group creates a new router group. You should add all the routes that have common middlewares or the same path prefix.
// For example, all the routes that use a common middleware for authorization could be grouped.
// 返回的依然是一个RouterGroup，但，是一个新的RouterGroup对象
// 且，Handlers和basePath都基于base group做了相应的更新
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
    return &RouterGroup{
        Handlers: group.combineHandlers(handlers),
        basePath: group.calculateAbsolutePath(relativePath),
        engine:   group.engine,
    }
}
```

以上就做好了url与处理逻辑的绑定，下面再看一下是如何Run起来的

```go
  // Run attaches the router to a http.Server and starts listening and serving HTTP requests.
  // It is a shortcut for http.ListenAndServe(addr, router)
  // Note: this method will block the calling goroutine indefinitely unless an error happens.
  func (engine *Engine) Run(addr ...string) (err error) {
      defer func() { debugPrintError(err) }()

      address := resolveAddress(addr)
      debugPrint("Listening and serving HTTP on %s\n", address)
      err = http.ListenAndServe(address, engine)
      return
  }
```
从上面看，就是直接调用了http包的ListenAndServe，嗯，那engine一定是实现了http要求的接口了

```go
  // ServeHTTP conforms to the http.Handler interface.
  func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
      // 从池子中取出一个ctx
      c := engine.pool.Get().(*Context)
      c.writermem.reset(w)
      // 请求放在了ctx的Request字段上
      c.Request = req
      c.reset()

      engine.handleHTTPRequest(c)

      engine.pool.Put(c)
  }
```
可以看到，只要实现这一个方法就能无缝对接http包的Server功能，完全不用care服务端的listen、accept、connect这些细节，只关注具体的处理逻辑就行了，这太值了！

从`ServeHTTP`的代码看出，这里的逻辑也很简略，就是从对象池取出一个Context，然后把它扔给`handleHTTPRequest`而已。关于Context的实现我们待会再细说，现在先重点分析下`handleHTTPRequest`相关的逻辑，上面设置了那么多middleware和handler，那么具体是怎么被执行的呢？

```go
  func (engine *Engine) handleHTTPRequest(c *Context) {
      // 解析出method、path
      httpMethod := c.Request.Method
      rPath := c.Request.URL.Path
      unescape := false
      if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
          rPath = c.Request.URL.RawPath
          unescape = engine.UnescapePathValues
      }
      rPath = cleanPath(rPath)

      // Find root of the tree for the given HTTP method
      // 然后开始在路由树中匹配
      t := engine.trees
      for i, tl := 0, len(t); i < tl; i++ {
          if t[i].method != httpMethod {
              continue
          }
          root := t[i].root
          // Find route in tree
          // 如果能匹配上，则会返回这个路径对应的handlers
          handlers, params, tsr := root.getValue(rPath, c.Params, unescape)
          if handlers != nil {
              // 注意全都赋给了Context
              // 艰巨的任务交给了Context同志
              c.handlers = handlers
              c.Params = params
              // 关键点，只是调用了一下Next()！！
              c.Next()
              c.writermem.WriteHeaderNow()
              return
          }
          if httpMethod != "CONNECT" && rPath != "/" {
              if tsr && engine.RedirectTrailingSlash {
                  redirectTrailingSlash(c)
                  return
              }
              if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
                  return
              }
          }
          break
      }
      
      // 路径匹配但方法不匹配的情况
      if engine.HandleMethodNotAllowed {
          for _, tree := range engine.trees {
              if tree.method == httpMethod {
                  continue
              }
              if handlers, _, _ := tree.root.getValue(rPath, nil, unescape); handlers != nil {
                  c.handlers = engine.allNoMethod
                  serveError(c, http.StatusMethodNotAllowed, default405Body)
                  return
              }
          }
      }
      // 没找到路径的情况
      c.handlers = engine.allNoRoute
      serveError(c, http.StatusNotFound, default404Body)
  }
```

那么问题来了，`c.Next()`是何方神圣？尽管说要等下再分析Context，但，已经等不及了..

```go
  // Next should be used only inside middleware.
  // It executes the pending handlers in the chain inside the calling handler.
  // See example in GitHub.
  func (c *Context) Next() {
      c.index++
      // 注意c.index初始化时是被置为-1的，因此
      // 逻辑也就是把c.handlers从头遍历、执行一遍
      for c.index < int8(len(c.handlers)) {
          c.handlers[c.index](c)
          c.index++
      }
  }
```
擦，竟然丧心病狂的只有这么短？而且逻辑也很清楚，就是把handlers依次执行一遍，这能实现普通的handler的功能我信，但能实现middleware的功能我是不信的，顺便科普下大名鼎鼎的middleware的基本功能：

> I use the term middleware, but each language/framework calls the concept differently. NodeJS and Rails calls it middleware. In the Java enterprise world (i.e. Java Servlet), it’s called filters. C# calls it delegate handlers. **Essentially, the middleware performs some specific function on the HTTP request or response at a specific stage in the HTTP pipeline before or after the user defined controller.** Middleware is a design pattern to eloquently add cross cutting concerns like logging, handling authentication, or gzip compression without having many code contact points. refer: [What is HTTP middleware?](https://www.moesif.com/blog/engineering/middleware/What-Is-HTTP-Middleware/#)

一个http中间件基本上会分别在用户自定义业务逻辑之前和之后执行，所以一个含有中间件的业务的执行逻辑应该是这样：

- 中间件的pre_request逻辑
- 用户自定义业务逻辑
- 中间件的post_request逻辑

可以看到中间件的逻辑是被用户自定义逻辑给割裂开了，分两段执行，用一张图来表示可能更清楚：

![](https://i.imgur.com/96NE89V.png)

直接找一个内置middleware的实现来分析下：

```go
// LoggerWithConfig instance a Logger middleware with config.
func LoggerWithConfig(conf LoggerConfig) HandlerFunc {
        
        // ...
        // 前面省略一堆配置logger的逻辑
        // ...
    
        return func(c *Context) {
        // Start timer
        start := time.Now()
        path := c.Request.URL.Path
        raw := c.Request.URL.RawQuery

        // Process request
        // 这里也调了c.Next()！！
        c.Next()

        // Log only when path is not being skipped
        if _, ok := skip[path]; !ok {
            param := LogFormatterParams{
                Request: c.Request,
                isTerm:  isTerm,
                Keys:    c.Keys,
            }

            // Stop timer
            param.TimeStamp = time.Now()
            param.Latency = param.TimeStamp.Sub(start)

            param.ClientIP = c.ClientIP()
            param.Method = c.Request.Method
            param.StatusCode = c.Writer.Status()
            param.ErrorMessage = c.Errors.ByType(ErrorTypePrivate).String()

            param.BodySize = c.Writer.Size()

            if raw != "" {
                path = path + "?" + raw
            }

            param.Path = path

            fmt.Fprint(out, formatter(param))
        }
    }
}
```

可以看到，玄机在于middleware里也调了一次`c.Next()`，那为什么这样就能实现middleware的功能呢？我们来分析一下：

假设除了用户业务逻辑外，只有这一个middleware，那么，执行流程会是这样：

- `handleHTTPRequest` 中的`c.Next()`触发了Logger middleware的执行
- Logger middleware先执行了直到`c.Next()`之前的逻辑
- 又进入`c.Next()`函数，此时index++，开始下一个handler函数，即用户自定义业务逻辑
- 执行用户自定义业务逻辑完成后返回
- 此时，第二步中的`c.Next()`调用执行完成
- 开始执行`c.Next()`后边的逻辑
- 完成

可以看到，里面涉及到了对`c.Next()`的递归调用，而middleware功能的实现正是借助对函数的递归调用和层层返回来实现的。

middleware的实现逻辑理清楚后我们来正式看下Context的实现，这个号称是Gin里最重要的结构：

```go
  // Context is the most important part of gin. It allows us to pass variables between middleware,
  // manage the flow, validate the JSON of a request and render a JSON response for example.
  type Context struct {
      writermem responseWriter
      Request   *http.Request
      Writer    ResponseWriter

      Params   Params
      handlers HandlersChain
      index    int8

      engine *Engine

      // Keys is a key/value pair exclusively for the context of each request.
      Keys map[string]interface{}

      // Errors is a list of errors attached to all the handlers/middlewares who used this context.
      Errors errorMsgs

      // Accepted defines a list of manually accepted formats for content negotiation.
      Accepted []string
  }
```
通过注释我们看到，它的作用有：流程控制（即实现middleware）、校验和绑定request和渲染response。流程控制我们前面已经分析过了，下面重点分析下后两个是怎么做的

先分析request的处理，先来个例子看下怎么用的：

```go
// Binding from JSON
type Login struct {
	User     string `form:"user" json:"user" xml:"user"  binding:"required"`
	Password string `form:"password" json:"password" xml:"password" binding:"required"`
}

func main() {
	router := gin.Default()

	// Example for binding JSON ({"user": "manu", "password": "123"})
	router.POST("/loginJSON", func(c *gin.Context) {
		var json Login
		if err := c.ShouldBindJSON(&json); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		
		if json.User != "manu" || json.Password != "123" {
			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			return
		} 
		
		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
	})
	
    // Listen and serve on 0.0.0.0:8080
	router.Run(":8080")
}
```

可以看到它通过一个`c.ShouldBindJSON`可以把request data（json格式的）绑定到自定义的struct上，它还支持很多其它方法来绑定其它格式的输入，比如：

- ShouldBindXML
- ShouldBindQuery
- ShouldBindYAML
- ShouldBindHeader

我们来具体分析下它的实现原理，拿一个`BindJSON`来看，其它的都类似

```go
  // BindJSON is a shortcut for c.MustBindWith(obj, binding.JSON).
  func (c *Context) BindJSON(obj interface{}) error {
      return c.MustBindWith(obj, binding.JSON)
  }
  
  // MustBindWith binds the passed struct pointer using the specified binding engine.
  // It will abort the request with HTTP 400 if any error occurs.
  // See the binding package.
  func (c *Context) MustBindWith(obj interface{}, b binding.Binding) error {
      if err := c.ShouldBindWith(obj, b); err != nil {
          c.AbortWithError(http.StatusBadRequest, err).SetType(ErrorTypeBind) // nolint: errcheck
          return err
      }
      return nil
  }
  
  // ShouldBindWith binds the passed struct pointer using the specified binding engine.
  // See the binding package.
  func (c *Context) ShouldBindWith(obj interface{}, b binding.Binding) error {
      return b.Bind(c.Request, obj)
  }
  
  // Binding describes the interface which needs to be implemented for binding the
  // data present in the request such as JSON request body, query parameters or
  // the form POST.
  type Binding interface {
      Name() string
      Bind(*http.Request, interface{}) error
  }
  
  // These implement the Binding interface and can be used to bind the data
  // present in the request to struct instances.
  var (
      JSON          = jsonBinding{}
      XML           = xmlBinding{}
      Form          = formBinding{}
      Query         = queryBinding{}
      FormPost      = formPostBinding{}
      FormMultipart = formMultipartBinding{}
      ProtoBuf      = protobufBinding{}
      MsgPack       = msgpackBinding{}
      YAML          = yamlBinding{}
      Uri           = uriBinding{}
  )
  
type jsonBinding struct{}

func (jsonBinding) Name() string {
    return "json"
}

func (jsonBinding) Bind(req *http.Request, obj interface{}) error {
    if req == nil || req.Body == nil {
        return fmt.Errorf("invalid request")
    }
    return decodeJSON(req.Body, obj)
}

func decodeJSON(r io.Reader, obj interface{}) error {
    decoder := json.NewDecoder(r)
    if EnableDecoderUseNumber {
        decoder.UseNumber()
    }
    if err := decoder.Decode(obj); err != nil {
        return err
    }
    return validate(obj)
}
```

可以看到，`BindXXX` --> `MustBindWith` --> `ShouldBindWith` --> `b.Bind`，所有支持的格式都是一样的模式，后续如果想增加一个新的格式，只需要按照`Binding`
这个接口来实现一下那两个方法就可以用了。

对参数的校验用到了[validator](https://github.com/go-playground/validator)这个包，下面具体分析一下

以 bindJSON 为例，最终会调用到`decodeJSON`：

```go
func decodeJSON(r io.Reader, obj interface{}) error {
	decoder := json.NewDecoder(r)
	if EnableDecoderUseNumber {
		decoder.UseNumber()
	}
	if EnableDecoderDisallowUnknownFields {
		decoder.DisallowUnknownFields()
	}
	if err := decoder.Decode(obj); err != nil {
		return err
	}
	return validate(obj)
}

var Validator StructValidator = &defaultValidator{}

// validate的定义在此
// 位于binding/binding.go
func validate(obj interface{}) error {
	if Validator == nil {
		return nil
	}
	return Validator.ValidateStruct(obj)
}

// 位于binding/default_validator.go
func (v *defaultValidator) ValidateStruct(obj interface{}) error {
	value := reflect.ValueOf(obj)
	valueType := value.Kind()
	if valueType == reflect.Ptr {
		valueType = value.Elem().Kind()
	}
	if valueType == reflect.Struct {
	   // 懒加载
		v.lazyinit()
		if err := v.validate.Struct(obj); err != nil {
			return err
		}
	}
	return nil
}

func (v *defaultValidator) lazyinit() {
	v.once.Do(func() {
		v.validate = validator.New()
		v.validate.SetTagName("binding")
	})
}
```


下面分析下处理完的结果是如何返回http response的（包括http status、header、resp body等）：

我们知道`net/http`包中经典函数签名`ServeHTTP(w http.ResponseWriter, req *http.Request)`，其中这个w就是用于返回response的，向它写入数据即可，那么Gin中是怎么跟它联系上的呢？

仍然需要再看一遍engine的ServeHTTP这个方法

```go
  // ServeHTTP conforms to the http.Handler interface.
  func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
      // 从池子中取出一个ctx
      c := engine.pool.Get().(*Context)
      // 此处把w绑定到ctx上了
      c.writermem.reset(w)
      // 请求放在了ctx的Request字段上
      c.Request = req
      // 设置Writer等, 重置handlers，后面handleHTTPRequest会根据请求路径重新计算
      c.reset()

      engine.handleHTTPRequest(c)

      engine.pool.Put(c)
  }
  // 来自：response_writer.go
  type responseWriter struct {
    http.ResponseWriter
    size   int
    status int
}
  // 来自：response_writer.go
  func (w *responseWriter) reset(writer http.ResponseWriter) {
    // w是在这里被绑定到Gin内部的一个结构体responseWriter上的
    w.ResponseWriter = writer
    w.size = noWritten
    w.status = defaultStatus
}
   // 来自：context.go
  func (c *Context) reset() {
      c.Writer = &c.writermem
      c.Params = c.Params[0:0]
      c.handlers = nil
      // 初始是-1，那就可以理解Next()中的index++了
      c.index = -1
      c.Keys = nil
      c.Errors = c.Errors[0:0]
      c.Accepted = nil
  }
```
思考一个问题：

> 为何有`writermem`和`Writer`两个功能类似的字段？它们两个从功能上看都是用于返回response的

使用上看，`http.ResponseWriter`先被赋给了`writermem`里的一个字段，然后`writermem`又被赋给了`Writer`。那么为啥要这么绕呢？我的理解：`Writer`是gin自己定义的接口，它扩展了Golang的`http.ResponseWriter`接口，因此直接把`http.ResponseWriter`赋给`Writer`肯定不行，只能曲线救国，这样就可以既能享用`http.ResponseWriter`的好处又能有自己的扩展功能。（其实，感觉只使用`writermem`这一个字段也能满足，暂时没想明白增加一个自定义接口的作用）

返回的response的格式也可以是多种多样的，比如json、xml、yaml等，是如何支持多种格式的呢？以json为例：

```go
  // JSON serializes the given struct as JSON into the response body.
  // It also sets the Content-Type as "application/json".
  func (c *Context) JSON(code int, obj interface{}) {
      c.Render(code, render.JSON{Data: obj})
  }
  
  // Render writes the response headers and calls render.Render to render data.
  func (c *Context) Render(code int, r render.Render) {
      c.Status(code)

      if !bodyAllowedForStatus(code) {
          r.WriteContentType(c.Writer)
          c.Writer.WriteHeaderNow()
          return
      }

      if err := r.Render(c.Writer); err != nil {
          panic(err)
      }
  }
  
  // Status sets the HTTP response code.
  func (c *Context) Status(code int) {
      c.writermem.WriteHeader(code)
  }
  
// Render interface is to be implemented by JSON, XML, HTML, YAML and so on.
type Render interface {
    // Render writes data with custom ContentType.
    Render(http.ResponseWriter) error
    // WriteContentType writes custom ContentType.
    WriteContentType(w http.ResponseWriter)
}
  
  // JSON contains the given interface object.
  // 来源：render/json.go
  // 这就是上面那个render.JSON
  type JSON struct {
      Data interface{}
  }
  
  // Render (JSON) writes data with custom ContentType.
  // 来源：render/json.go
  func (r JSON) Render(w http.ResponseWriter) (err error) {
      if err = WriteJSON(w, r.Data); err != nil {
          panic(err)
      }
      return
  }
  
  // WriteJSON marshals the given interface object and writes it with custom ContentType.
  // 来源：render/json.go
  func WriteJSON(w http.ResponseWriter, obj interface{}) error {
      writeContentType(w, jsonContentType)
      jsonBytes, err := json.Marshal(obj)
      if err != nil {
          return err
      }
      _, err = w.Write(jsonBytes)
      return err
  }
```

跟绑定request是一样的套路，`c.XXX` --> `c.Render` --> `r.Render(c.Writer)`，后续如果想加新的格式，遵循Render接口实现那两个方法即可

### 其它

#### 对json的优化

内置了两种json实现，一个是标准库的`encoding/json`，另一个是`github.com/json-iterator/go`，在编译的时候通过指定tag可以选择使用哪种实现。我们看看这是怎么做到的

定义好内部包`internal/json`，这个包下有两个文件，json.go和jsoniter.go，分别代表标准库和jsoniter的实现，以jsoniter为例来看一下：

```go
// +build jsoniter

package json

import "github.com/json-iterator/go"

var (
    json = jsoniter.ConfigCompatibleWithStandardLibrary
    // Marshal is exported by gin/json package.
    Marshal = json.Marshal
    // Unmarshal is exported by gin/json package.
    Unmarshal = json.Unmarshal
    // MarshalIndent is exported by gin/json package.
    MarshalIndent = json.MarshalIndent
    // NewDecoder is exported by gin/json package.
    NewDecoder = json.NewDecoder
    // NewEncoder is exported by gin/json package.
    NewEncoder = json.NewEncoder
)
```

只是import相应的包，然后再把一些公开方法export出来，注意到最开头的一行：`// +build jsoniter`，它的意思是，当指定编译tag为`jsoniter`时，使用当前文件来编译

对比另一个文件， 它的开头写着：`// +build !jsoniter`，表示当tag不是`jsoniter`时用这个文件编译

```go
// +build !jsoniter

package json

import "encoding/json"

var (
    // Marshal is exported by gin/json package.
    Marshal = json.Marshal
    // Unmarshal is exported by gin/json package.
    Unmarshal = json.Unmarshal
    // MarshalIndent is exported by gin/json package.
    MarshalIndent = json.MarshalIndent
    // NewDecoder is exported by gin/json package.
    NewDecoder = json.NewDecoder
    // NewEncoder is exported by gin/json package.
    NewEncoder = json.NewEncoder
)
```

然后，在使用的地方import内部封装的包：`github.com/gin-gonic/gin/internal/json`

最后，当编译时需指定tag：`$ go build -tags=jsoniter .`

[Go语言中自动选择json解析库](https://www.flysnow.org/2017/11/05/go-auto-choice-json-libs.html)

### 参考

- [拆轮子系列：gin框架](https://juejin.im/post/5b38a6b16fb9a00e6714ab3a)
- [Golang gin框架源码解析](http://www.itarea.cn/golang/57.html)
- [从 net/http 入门到 Gin 源码梳理](https://xguox.me/gin-source-code.html/)
- [gin 源码剖析](https://github.com/hhstore/blog/issues/131)
