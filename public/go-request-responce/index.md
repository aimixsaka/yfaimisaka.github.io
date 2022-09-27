# Go中的请求和响应


## $ 前言

记录一下`go`中的请求和响应处理

## $ 请求

### $ Request结构

- `Request`结构既可以用于客户端也可以用于服务端

- `Request`结构

  ```go
  type Request struct {
      // Method specifies the HTTP method (GET, POST, PUT, etc.).
      Method string
      
      URL *url.URL /*#*/
      
      Proto      string // "HTTP/1.0"
      ProtoMajor int    // 1
      ProtoMinor int    // 0
      
      // Request Header
      Header Header /*#*/
      
      // Request Body
      Body io.ReadCloser /*#*/
      
      // return a new copy of Body
      // used for client requests
      // For server requests, it is unused
      GetBody func() (io.ReadCloser, error)
      
      // value -1 indicates that the length is unknown
      // Values >= 0 indicate that the given number of bytes may
  	// be read from Body
      ContentLength int64
      
      TransferEncoding []string
      
      // Close indicates whether to close the connection after
  	// replying to this request (for servers) or after sending this
  	// request and reading its response (for clients).
  	//
  	// For server requests, the HTTP server handles this automatically
  	// and this field is not needed by Handlers.
  	//
  	// For client requests, setting this field prevents re-use of
  	// TCP connections between requests to the same hosts, as if
  	// Transport.DisableKeepAlives were set.
      Close bool
      
      // thisis either the value of the "Host" header 
      // or the host namegiven in the URL itself
      Host string
      
      // including both the URL field's query parameters
      // and the PATCH, POST, or PUT form data
      Form url.Values /*#*/
      // PostForm contains the parsed form data from PATCH, POST
  	// or PUT body parameters
      PostForm url.Values /*#*/
      // MultipartForm is the parsed multipart form, including file uploads
      MultipartForm *multipart.Form /*#*/
      
      // Trailer specifies additional headers 
      // that are sent after the request body
      Trailer Header
      
      // RemoteAddr allows HTTP servers and other software to record
  	// the network address that sent the request
      RemoteAddr string
      
      RequestURI string
      
      TLS *tls.ConnectionState
      
      Cancel <-chan struct{}
      
      // Response is the redirect response which caused this request
  	// to be created. This field is only populated during client
  	// redirects.
  	Response *Response
  }
  
  ```

#### $ URL字段

- `URL`结构体

  ```go
  type URL struct {
      Scheme   string
      Opaque   string
      User     *Userinfo
      Host     string
      Path     string
      RawQuery string
      Fragment string
  }
  ```

  

- `URL`的一般结构

  > scheme://[ userinfo@]host/path\[$query][#fragment]

#### $ Header字段

- 一个 首部就是一个映射， 这个映射的键为字符串， 值为字符串切片


##### $ 获取和设置或修改Header中字段

- 1. 通过切片访问`Header`中的特定字段，返回切片

     `w.Header["Accept-Encodeing"]`

     结果：`[gzip, deflate]`

- 2. 通过`Get`和`Set`方法，`Get`返回逗号分隔的字符串

     `w.Header.Get("Accept-Encodeing")`

     结果：`gzip, deflate`

#### $ Body字段

- `Body`是一个`ReadCloser `接口， 该接口既包含了`Reader`接口，也包含了`Closer`接口

- 一个例子

  ```go
  package main
  
  import (
      "fmt"
      "net/http"
  )
  
  func body(w http.ResponseWriter, r *http.Request) {
      len := r.ContentLength
      body := make([]byte, len)
      r.Body.Read(body)
      fmt.Fprintln(w, string(body))
  }
  
  func main() {
      server := http.Server{
          Addr: "127.0.0.1:8080",
      }
      http.HandleFunc("/body", body)
      server.ListenAndServe()
  }
  ```

  

### $ Go与HTML表单

#### $ HTML表单的一般形式

```html
<form action="/process" method="post" enctype="application/x-www-form-urlencoded">
    <input type="text" name="first_name"/>
    <input type="text" name="last_name"/>
    <input type="submit"/>
</form>
```

#### $ 数据编码方式

- `application/x-www-form-urlencoded`

  将HTML中表单中的数据编码为连续的**长查询字符串**

- `multipart/form-data`

  将表单中的数据转换为一条`MIME报文`，表单中的每个键值对都构成了这条报文的一部分

- `text/plain`

  HTML5支持

> 如何选择 $
>
> - `application/x-www-form-urlencoded` （简单文本数据
> - `multipart/form-data` （传输大量数据，如上传文件
> - `text/plain` （``Base64`编码，文本方式传输二进制数据

### $ 表单的解析

#### $ 手动解析

> `application/x-www-form-urlencoded`编码

- 1. 先调用`w.ParseForm()`方法

- 2. 通过`w.Form`字段访问表单数据和URL参数
- 3. 或者通过`w.PostForm`字段实现只访问表单数据

> `multipart/form-data`编码

- 1. 先调用`w.ParseMultipartForm()`方法
- 2. 通过`w.MultipartForm`字段访问表单数据或获取上传的文件

#### $ 自动解析

- `FormValue(key string)`
- `PostFormValue(key string)`

这两个方法会自动调用`ParseForm()和ParseMultipartForm()`方法进行解析

> 注：如果对应该键的值不止一个，方法只返回第一个值

### $ 文件上传和接收

**举个例子**

- HTML表单

  ```html
  <html> 　
    <head> 　 　
      <meta http-equiv="Content-Type" content=" text/html; charset=utf-8"/> 　 　
      < title>Go Web Programming</title> 
    </head> 　
    <body> 　 
      <form action="http://localhost:8080/process$hello=world&thread=123" method="post" enctype="multipart/form-data"> 　 　
         <input type="text" name="hello" value="sau sheong"/> 　 　 
         <input type="text" name="post" value="456"/> 　 　 　
         <input type="file" name="uploaded"/>
         <input type="submit"/> 
       </form> 　
     </body> 
  </html>
  ```

  

- Go处理

  ```go
  package main
  
  import(
      "fmt"
      "io/util"
      "net/http"
  )
  
  func process(w http.ResponseWriter, r *http.Request) {
      r.ParseMultipartForm(1024)
      fileHeader := r.MultipartForm.File("uploaded")[0]
      file, err := fileHeader.Open()
      if err == nil {
          data, err := ioutil.ReadAll(file)
          if err == nil {
              fmt.Fprintln(w, string(data))
          }
      }
  }
  
  func main() {
      server := http.Server{
          Addr: "127.0.0.1"
      }
      http.HandleFunc("/process", process)
      server.ListenAndServe()
  }
  ```



**或者用`FormFile()`方法**

```go
func process(w http.ResponseWriter, r *http.Request) {
    file, _, err := r.FormFile("uploaded")
    if err == nil {
        data, err := ioutil.ReadAll(file)
        if err == nil {
            fmt.Fprintln(w, string(data))
        }
    }
}
```

当文件只有一个时这特别有用
