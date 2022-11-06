---
title: Protoc使用
tags: 
  - Golang
  - Protoc
  - Goland
date: 2022-08-24
categories:
    - 工具
---

### 环境
- 系统: windows
- 语言: golang
- IDE: goland

### [安装 Protoc](https://github.com/protocolbuffers/protobuf/releases)
- 下载`protoc-xxx-win64.zip`, 并解压到安装位置(如: `d:/protoc`)即可
- 添加环境变量`d:/protoc/bin`到`PATH`中
- 使用`protoc --version`验证是否安装成功

### 配置 Golang 环境
- 下载第三方`protobuf`库, 并解压到`d:/protoc/include`, 如:
    - [googleapis](https://github.com/googleapis/googleapis)
    - [gogo/protobuf](https://github.com/gogo/protobuf)
- `go install google.golang.org/protobuf/cmd/protoc-gen-go`
- 安装`goland`插件 [Protocol Buffer Editor](https://github.com/jvolkman/intellij-protobuf-editor), 按照`README`在配置中添加`d:/protoc/include`和自己实现的库的路径(目前好像是每个项目使用时都需要配置一下), 完成之后就有自动提示了（可能需要重启IDE）

### 使用
- 命令: `protoc -I=. --go_out=. example.proto`
    - `-I`用于指定`imports path`, 第三方库统一放到`d:/protoc/include`的话是不需要指定的

### 官方文档
包括protobuf语法和其他语言的使用教程[点击](https://developers.google.com/protocol-buffers/docs/overview)

