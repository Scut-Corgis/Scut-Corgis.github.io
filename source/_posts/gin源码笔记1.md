---
title: gin源码笔记1
date: 2024-09-10 23:30:54
toc: true
tags:
    - gin
---
# 前言

主要是通过gin的源码，学习一些golang编程的常见技巧。同时积累标准库的用法。

# options模式
一种设置结构体参数的方法，用于设置默认值，但用户可以传入option来修改默认值，具备可扩展性。

1. gin初始化时会传入opts
``` go
func New(opts ...OptionFunc) *Engine {
    engine := &Engine{
        // 默认参数
        ...
    }
    return engine.With(opts...)
}
```
2. options应用
``` go
func (engine *Engine) With(opts ...OptionFunc) *Engine {
    for _, opt := range opts {
        opt(engine)
    }

    return engine
}
```
3. 如可以写一个OptionFunc并加载
``` go
func WithForwardedByClientIP() gin.OptionFunc {
    return func(engine *gin.Engine) {
        engine.ForwardedByClientIP = true
    }
}

r := gin.Default(WithForwardedByClientIP())
```

# 中间件扩展原理

1. 插入中间件，实际上将中间件对应的函数闭包`HanlerFunc`存到路由切片里
``` go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
    engine.RouterGroup.Use(middleware...)
    ...
    return engine
}
// Use adds middleware to the group, see example code in GitHub.
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
    group.Handlers = append(group.Handlers, middleware...)
    return group.returnObj()
}
```
2. 以默认两个中间件为例，可以看到中间件`HandlerFunc`会操作`gin.Context`变量。调用`c.Next()`前为中间件前处理，调用后为请求完毕后处理。如日志中间件会记录调用耗时并打印。
``` go
engine.Use(Logger(), Recovery())
func Logger() HandlerFunc {
    return LoggerWithConfig(LoggerConfig{})
}
// LoggerWithConfig instance a Logger middleware with config.
func LoggerWithConfig(conf LoggerConfig) HandlerFunc {
    ...
    return func(c *Context) {
        // Start timer
        start := time.Now()
        path := c.Request.URL.Path
        raw := c.Request.URL.RawQuery

        // Process request
        c.Next()
        ...
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
```
3. `Next()`函数会执行后面的中间件
``` go
func (c *Context) Next() {
    c.index++
    for c.index < int8(len(c.handlers)) {
        c.handlers[c.index](c)
        c.index++
    }
}
```

4. 假如绑定了一个ping，当一个ping到达gin时，处理过程类似洋葱模型。跟深度优先遍历算法代码基本一样
![](gin洋葱模型.jpg)
``` go
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "message": "pong",
        })
    })
    r.Run() 

func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle(http.MethodGet, relativePath, handlers)
}
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    absolutePath := group.calculateAbsolutePath(relativePath)
    handlers = group.combineHandlers(handlers)
    group.engine.addRoute(httpMethod, absolutePath, handlers)
    return group.returnObj()
}
// 这里将所有handler汇总，返回最终的HandlersChain
func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
    finalSize := len(group.Handlers) + len(handlers)
    assert1(finalSize < int(abortIndex), "too many handlers")
    mergedHandlers := make(HandlersChain, finalSize)
    copy(mergedHandlers, group.Handlers)
    copy(mergedHandlers[len(group.Handlers):], handlers)
    return mergedHandlers
}
```

``` go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
    ...

    // 将handlers挂在前缀树对应path的叶子node上
    root := engine.trees.get(method)
    if root == nil {
        root = new(node)
        root.fullPath = "/"
        engine.trees = append(engine.trees, methodTree{method: method, root: root})
    }
    root.addRoute(path, handlers)

    if paramsCount := countParams(path); paramsCount > engine.maxParams {
        engine.maxParams = paramsCount
    }

    if sectionsCount := countSections(path); sectionsCount > engine.maxSections {
        engine.maxSections = sectionsCount
    }
}
```

5. 请求到达，取出method树对应的节点挂着的`hanlders`，赋值给Context，然后调用`Next()`，开始像洋葱模式一样执行。

``` go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // 取一个gin.Context
    c := engine.pool.Get().(*Context)
    c.writermem.reset(w)
    c.Request = req
    c.reset()
    // 实际处理
    engine.handleHTTPRequest(c)

    engine.pool.Put(c)
}
```

``` go
func (engine *Engine) handleHTTPRequest(c *Context) {
    httpMethod := c.Request.Method
    rPath := c.Request.URL.Path
    unescape := false
    ...
    // Find root of the tree for the given HTTP method
    t := engine.trees
    for i, tl := 0, len(t); i < tl; i++ {
        // 找到对应的方法树
        if t[i].method != httpMethod {
            continue
        }
        root := t[i].root
        // Find route in tree
        value := root.getValue(rPath, c.params, c.skippedNodes, unescape)
        if value.params != nil {
            c.Params = *value.params
        }

        if value.handlers != nil {
            // 这里最重要，取出了对应节点的hanlders赋给Context
            c.handlers = value.handlers
            c.fullPath = value.fullPath
            // 调用Next执行请求
            c.Next()
            c.writermem.WriteHeaderNow()
            return
        }
        if httpMethod != http.MethodConnect && rPath != "/" {
            if value.tsr && engine.RedirectTrailingSlash {
                redirectTrailingSlash(c)
                return
            }
            if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
                return
            }
        }
        break
    }

    if engine.HandleMethodNotAllowed {
        for _, tree := range engine.trees {
            if tree.method == httpMethod {
                continue
            }
            if value := tree.root.getValue(rPath, nil, c.skippedNodes, unescape); value.handlers != nil {
                c.handlers = engine.allNoMethod
                serveError(c, http.StatusMethodNotAllowed, default405Body)
                return
            }
        }
    }
    c.handlers = engine.allNoRoute
    serveError(c, http.StatusNotFound, default404Body)
}
```

# 标准库函数积累

``` go
// 测试format是否以 \n 结尾。
strings.HasSuffix(format, "\n")
// 往DefaultWriter里面写格式化string
fmt.Fprintf(DefaultWriter, "[GIN-debug] "+format, values...)
// 获取操作系统环境变量
os.Getenv("TERM") == "dumb"
// 将headers []string，字符串以\r\n拼接成一个string，如 sss\r\nbbb
strings.Join(headers, "\r\n")
```

## 对象池sync.Pool
sync.Pool 是 Go 语言标准库中的一个并发安全的内存池，主要用于临时对象的存储和复用，从而减少频繁的内存分配和垃圾回收，提高性能。

sync.Pool 主要用于存储短期使用的对象，特别是在高并发场景下，能够有效地减少内存分配的开销。它的设计适合于那些生命周期短暂的对象，比如在请求处理中使用的临时数据结构。

``` go
    // 分配Context的函数
    engine.pool.New = func() any {
        return engine.allocateContext(engine.maxParams)
    }
    // 取
    c := engine.pool.Get().(*Context)
    ...
    // 释放
    engine.pool.Put(c)
```