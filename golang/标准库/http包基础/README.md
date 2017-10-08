# 标准库net/http包使用及工作原理

## 一. 1个 HTTP Server的基本构成
一个Web应用从 `浏览器向服务器发送请求（Request）开始，到服务器根据请求做出相应响应（Response）结束`，整个流程中服务器的`控制核心`无疑是：服务器从HTTP Request中提取请求路径（URL）并找到对应的处理程序（Handler）处理请求，最后返回结果。

我们知道HTTP协议是基于TCP协议的，我们当然可以自己通过标准库net中提供TCP API开始写一个HTTP Server，但你不得不自己额外做的事有：
1. 实现TCP Server监听，为每一新来的TCP link建立一个goroutine, 并在goroutine与客户端交互（不用担心`单机C10K问题`，因为goroutine是用户态线程，很轻量级，可以很随意就创建成千上万个）。
2. 在每个goroutine中将TCP link中的请求数据按HTTP协议格式解析出来(可以将数据解析出成Request对象，以后的访问提供方便)，并根据其URL找到相应的处理程序Handler, 因此你还需要提前建立好 `URL：Handler映射表`。
3. 当处理程序处理结束后，你还需要将处理结果数据按HTTP协议格式返回给客户端。代码大致如下：
```go
//建立 URL:Handler映射表
//注意：由于table会在不同goroutine中使用，因此真正环境中需要锁保护
var table = map[string]Handler {
    "/": rootHandler,
    "/login": loginHandler,
    ... 
} 

//Server监听
ln, _ := net.Listen("tcp", ":8080")
for {
    conn, _ := ln.Accept()
    go handleConn(conn)
}

//请求处理
func handleConn(conn net.Conn) {
    //1. 将请求成封装Request对象
    //2. 从table中查找相应处理程序
    //3. 将处理结果封装HTTP格式数据返回给客户端
}

```
然而这些需求在net/http包中都满足了，为什么不直接用？(Don't Repeat Yourself)<br/>
net/http包中几个重要的类型:<br/>
`http.ServeMux`: 建立URL:Handler映射表<br/>
`http.Server`: 运行HTTP Server<br/>
`http.Request`: 封装客户端HTTP请求数据<br/>
`http.ResponseWriter`: 用来构造服务器端HTTP响应数据<br/>
`http.Handler`: URL处理程序必须实现的接口<br/><br/>

## 二. 使用net/http搭建一个最简单的HTTP Server
```go
//step1. 建立 URL:Handler映射表
mux := http.NewServeMux()
mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, world")
})

//step2. 创建并运行HTTP server
server := http.Server{Addr: ":8080", Handler: mux}
log.Fatal(server.ListenAndServe())

```
然后我们打开浏览器在地址栏输入：http://localhost:8080<br>
服务器将返回：Hello, world<br/>
是不是so easy!!!<br/>

## 三. 第二部分代码内部工作原理
第二部分代码和第一部分工作原理基本一致。首先看看step1中的 mux（http.ServeMux）的定义：
```go
type ServeMux struct {
	mu    sync.RWMutex        //保护m
	m     map[string]muxEntry //URL:Handler映射表
	hosts bool
}
type muxEntry struct {
	explicit bool
	h        Handler
	pattern  string
}
```
很显然当我们调用mux.HandleFunc(...)的时候就是添加`URL:Handler`键值对到mux的map中。
那么`问题`来了，func(http.ResponseWriter,*http.Request)是个函数类型，而http.Handler是一个接口类型，二者是如何转换的？不妨看看 mux.HandleFunc：
```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	mux.Handle(pattern, HandlerFunc(handler))  //又调用了mux.Handle
}
```
而 mux.Handle 就比较简单了，就是将 func(http.ResponseWriter,*http.Request)转换为 http.Handler 然后放入mux的map中。

`HandlerFunc`(注意不要与 `HandleFunc` 函数混淆了)是个什么鬼？又有什么用？
```go
type HandlerFunc func(ResponseWriter, *Request) 
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```
其实就是一个实现了 `http.Handler 接口`的类型，该类型底层基础类型就是 `func(ResponseWriter, *Request)`，我们知道在go语言中除了 `指针与接口` 其他基础类型也是可以定义方法的，标准库定义这个一个类型，为的就是将 普通`func(ResponseWriter, *Request)` 适配到 `http.Handler接口`. 所以 `HandlerFunc` 这样的类又称为`适配器类`,这一做法很有用，一定要掌握住该技巧。

好了，现在看看step2的 `http.Server`，将 `监听地址` 和 step1中的 `http.ServeMux` 对象传递给了它，然后调用`server.ListenAndServe()`开始监听, 处理流程大致如下：
1. server监听到有新链接进来，创建一个goroutine来处理新链接
2. 在goroutine中，将请求和响应分别封装为 http.Request和http.ResponseWriter对象。然后用这两个对象作为函数参数调用 server.Handler.serveHTTP(...), 而server.Handler 即为我们传入的 `http.ServeMux` 对象，而http.ServeMux对象的serveHTTP方法，我们都没有碰过，里面到底做了什么？
3. http.ServeMux对象的serveHTTP方法做的事，其实就是根据 http.Request对象中的URL 在自己的map中查找对应的Handler（这个又是我们在step1中添加的），然后执行。

绕了一大圈，简单来说就是 每当有新请求进来，server都会为我们新建一个goroutine，并在其中根据请求URL调用 我们在创建server之前添加的 URL:Handler映射表（通过server中的http.Handler字段混入）中的相应URL的Handler.

`问题：为什么不在server中放置一个 URL:Handler 映射表？` <br/>
这样上面步骤2中就不用先绕到 server.Handler.serveHTTP(...)中，才能查找映射表了。这样的话`http.ServeMux` 对象也不需要了。 这样做从程序逻辑上讲没有问题，但将`http.Server`的逻辑弄得更复杂了，通过一个 `http.Handler中间层` 将 `URL路由功能从http.Server 解耦出来`，虽然理解起来有点绕，但各自的职责将更加清楚（`http.Server`就是只管HTTP Server中通用功能的部分，业务逻辑的不同处理都通过 `http.ServeMux` 来构建）<br/>


## 四. 更简单的代码写法
```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, world")
})
log.Fatal(http.ListenAndServe(":8080", nil))
```
为什么连 `http.ServeMux` 和 `http.Server`都没有用到？其实肯定是需要的，只是被封装了起来。大家都说golang很容易上手，这其实得益于 实现golang的团队 强大的封装抽象能力，将复杂留给了自己，简单易用性给了开发者。

其实这种写法是使用了 net/http包中定义的一个 全局`http.ServeMux`对象变量 `DefaultServeMux`, 然后在 http.ListenAndServe(":8080", nil) 函数中新构建了一个 `http.Server`对象，然后让server进入监听。比较巧妙地是 函数http.ListenAndServe的第二个参数，如果是nil，该server才会使用`DefaultServeMux`，否则使用新传入的`http.ServeMux`对象。



