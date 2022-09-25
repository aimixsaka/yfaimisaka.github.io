# Go Web编程


## ? 前言

最近看了下`Go Web编程`这本书，再去看了下源码，想对`net/http`包中的一些关键的函数，接口的定义和关系做一个记录

### ? 多路复用器

> Go中web应用需要对客户端的请求响应，为了将不同请求URL转发到不同的处理器(Handler)进行处理需要用到多路复用器(ServeMux)

- `(var) DefaultServeMux`: `ListenAndServe(string, Handler) error` 中第二个参数的默认值，实质是一个指向`ServeMux`类型的指针，是`ServeMux`的一个实例，但`ServeMux`同时又实现了Handler接口(故可以充当第二个参数的默认值)

  **源码:**

  ```go
  // DefaultServeMux is the default ServeMux used by Serve.
  var DefaultServeMux = &defaultServeMux
  
  var defaultServeMux ServeMux
  ```

  

- `(func) NewServeMux()`: 返回一个`ServeMux`

- `(struct) ServeMux`: 多路复用器结构，负责URL的路径和处理的映射，以及通过`自身的ServeHTTP`方法调用URL映射对应的`处理器的ServeHTTP方法`

  **源码:**

  ```go
  type ServeMux struct {
  	mu    sync.RWMutex
  	m     map[string]muxEntry
  	es    []muxEntry // slice of entries sorted from longest to shortest.
  	hosts bool       // whether any patterns contain hostnames
  }
  ```

  

### ?  服务器

> 负责启动监听和配置服务
>
> 如果简单配置可以直接调`http.ListenAndServe`函数来进行监听，复杂配置则需要构建一个Server结构体来调用其ListenAndServe方法

- `(struct) Server`: 用于细化配置的结构体

  **源码:**

  ```go
  type Server struct {
      Addr string
      Handler Handler
      
      TLSConfig *tls.Config
      
      ReadTimeout time.Duration
      ReadHeaderTimeout time.Duration
      WriteTimeout time.Duration
      IdleTimeout time.Duration
      
      MaxHeaderBytes int
      
      TLSNextProto map[string]func(*Server, *tls.Conn, Handler)
      
      ConnState func(net.Conn, ConnState)
      
      ErrorLog *log.Logger
      
      BaseContext func(net.Listener) context.Context
      ConnContext func(ctx context.Context, c net.Conn) context.Context
  }
  ```

- `(func) ListenAndServe(addr string, handler Handler) error`: 开启监听和进行处理，是整个web应用的启动器

- `(func) ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error`: ListenAndServe的https版本

### ?  处理器和处理器函数

> Go web中实际处理请求的函数或类型
>
> 处理器：实现了Handler接口的任意类型，可以是结构体，函数类型等
>
> 处理器函数：普通的函数，但是必须要以 `http.ResponseWriter`和`*http.Request`为参数



- `(interface) Handler {ServeHTTP(ResponseWriter, *Request)}`: 处理器接口，实现了该接口的即为处理器

- `(func) http.Handle(pattern string, handler Handler)`: 将Handler和DefaultServeMux (或`自定义的ServeMux`) 绑定，使其处理`pattern`上的请求。实质是`ServeMux`的方法，方便调用故在http包中创建。**实质:**

  ```go
  // http.Handle
  func Handle(pattern string, handler Handleer) {
      DefaultServeMux.Handle(pattern, handler)
  }
  
  // 在server.go源码中
  ```

  

- `(func) http.HandleFunc(pattern string, handler func(ResponseWriter, *Request))`: 将处理器函数与`ServeMux`绑定并监听`pattern`路径。实质会被转换为一个`Handler`:

  ```go
  // http.HandleFunc
  func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
      DefaultServeMux.HandleFunc(pattern, handler)
  }
  
  // ServeMux.HandleFunc
  func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
      mux.Handle(pattern, HandlerFunc(handler))
  } // 通过显式类型转换将底层类型相同的handler转换为HandlerFunc类型
  
  // HandlerFunc是个函数类型
  type HandlerFunc func(ResponseWriter, *Request)
  /*并且它实现了Handler接口，也就是说，
  HandlerFunc类型是个处理器(Handler),
  故调用HandleFunc方法实质是调用Handle方法，
  且将一个普通函数类型的参数转换成了一个处理器
  */
  func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
  }
  ```

    

  
