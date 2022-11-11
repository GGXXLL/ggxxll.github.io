---
title: gin+opentracing的使用
tags: 
  - Bug
  - Golang
  - Gin
date: 2022-08-24
categories:
  - 后端
---

## 环境
- 系统: windows
- 语言: golang
- 框架: gin

## Bug描述
在如下代码中, `ctx := c`和`ctx := context.Background()`传递到`cli.Set`中是没有问题的.

使用`ctx := c.Request.Context()`时, 会打印出错误: `context canceled`. 
```golang
package main

import (
    "fmt"
    "github.com/gin-gonic/gin"
    "github.com/go-redis/redis/v8"
)

func main() {

    cli := redis.NewClient(&redis.Options{
        Addr: "127.0.0.1:6379",
    })

    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        //ctx := c
        //ctx := context.Background()
        ctx := c.Request.Context()
        go func() {
            if err := cli.Set(ctx, "hello", 1, 0).Err(); err != nil {
                fmt.Println(err)
            }
        }()
        c.JSON(200, gin.H{
            "message": "pong",
        })
    })
    r.Run() // listen and serve on 0.0.0.0:8080
}
```

## 分析
gin的`Request`对象来自`*http.Request`, `handlerFunc`被调用完成之后, `context`就被`cancel`了

在`ctx := c.Request.Context()`打断点, 可以找到如下代码:
```golang
func{
    ...
    serverHandler{c.server}.ServeHTTP(w, w.req)
    w.cancelCtx()
    ...
}

```

## 其他
- 使用gin+opentracing时, 不能使用  作为上下文传递, 要使用`c.Request.Context()`, 而不是`gin.Context`.
- 需要在goroutine中传递ctx, 应该拿出span新建一个ctx
```golang
func NewTracingContextWithParentContext(ctx context.Context) context.Context {
    span := opentracing.SpanFromContext(ctx)
    return opentracing.ContextWithSpan(context.Background(), span)
}
```
- [ginhttp](github.com/opentracing-contrib/go-gin/ginhttp)中封装好了相应的中间件,
```golang
func Middleware(tr opentracing.Tracer, options ...MWOption) gin.HandlerFunc {
    opts := mwOptions{
        opNameFunc: func(r *http.Request) string {
            return "HTTP " + r.Method
        },
        spanObserver: func(span opentracing.Span, r *http.Request) {},
        urlTagFunc: func(u *url.URL) string {
            return u.String()
        },
    }
    for _, opt := range options {
        opt(&opts)
    }

    return func(c *gin.Context) {
        carrier := opentracing.HTTPHeadersCarrier(c.Request.Header)
        ctx, _ := tr.Extract(opentracing.HTTPHeaders, carrier)
        op := opts.opNameFunc(c.Request)
        sp := tr.StartSpan(op, ext.RPCServerOption(ctx))
        ext.HTTPMethod.Set(sp, c.Request.Method)
        ext.HTTPUrl.Set(sp, opts.urlTagFunc(c.Request.URL))
        opts.spanObserver(sp, c.Request)

        // set component name, use "net/http" if caller does not specify
        componentName := opts.componentName
        if componentName == "" {
            componentName = defaultComponentName
        }
        ext.Component.Set(sp, componentName)
        c.Request = c.Request.WithContext(
            opentracing.ContextWithSpan(c.Request.Context(), sp))

        c.Next()

        ext.HTTPStatusCode.Set(sp, uint16(c.Writer.Status()))
        sp.Finish()
    }
}
```