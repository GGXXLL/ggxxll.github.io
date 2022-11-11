---
title: 文件下载Bug
tags: 
  - Golang
date: 2022-08-24
categories:
  - 后端
---

## 环境
- golang

## 问题

在`k8s`中部署多个节点服务时，CDN的Range回源经常失败。

原因：每个节点生成渠道包的时间（Modify-Time）不同，导致CDN回源时认为文件变动了，自动停止回源。

解决：使用`http.ServeContent`,  手动传递

## 代码

```go
// serveFile
// 将newApkPath地址下的文件返回给请求方。
// 注意：曾经这里是使用ctx.File(调用了Go的http.ServeFile)方法返回文件，
// 但是这样做在集群部署的情况下存在问题。集群每个节点文件生成的时间存在差异，
// 而go标准库中的实现是根据文件的最后修改时间判断文件是否发生了变动。
// 大文件下载时CDN会采用分片回源（Range Header）， 每个分片获取到的最后修改时间不一致，会判定为下载失败！
// 这里改成http.ServeContent实现后，强制将修改时间置为0值，Go的标准库会忽略最后修改时间检测，在集群-多节点环境下也能正常分片下载了。
func serveFile(ctx *gin.Context, newApkPath string) error {
	_, apkName := filepath.Split(newApkPath)
	ctx.Header("Content-Type", "application/octet-stream")
	ctx.Header("Content-Disposition", "attachment; filename="+apkName)
	ctx.Header("Content-Transfer-Encoding", "binary")
	f, err := os.OpenFile(newApkPath, os.O_RDONLY, os.ModePerm)
	if err != nil {
		return err
	}
	defer f.Close()
    /* 最后修改时间，强制0值，跳过检查 */
	http.ServeContent(ctx.Writer, ctx.Request, apkName, time.Time{}, f)
	return nil
}

```

