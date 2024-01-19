## 初始化验证码生成服务的基础代码（初始化Kratos项目）

在 laomaDJ/backend 中构建。

推荐使用工具 kratos 工具完成项目创建：

```shell
$ kratos new verifyCode
```

该工具基于 [Kratos Layout](https://github.com/go-kratos/kratos-layout) 创建项目。这个布局是典型的 Kratos 推荐布局。在 Github 上，若出现网络问题，也可以使用 Gitee 上的仓库，[Kratos Layout](https://gitee.com/go-kratos/kratos-layout.git)：

```shell
# -r 指定源
$ kratos new verifyCode -r https://gitee.com/go-kratos/kratos-layout.git
```

该布局提供了最基本的目录结构和一个用于测试的 HTTP 和 gRPC 接口。

进入项目目录，拉取依赖：

```shell
$ cd verifyCode
$ go mod tidy
```

运行该项目前，需要先生成相应源码，主要是使用了 wire 的依赖注入相关代码：

```shell
$ go get github.com/google/wire/cmd/wire
$ go generate ./... 
递归生成项目中的一些特殊文件生成一些文件

# 输出结果：
'''
E:\zhoulearn\kratos\verifyCode>go generate ./...
wire: verifyCode/cmd/verifyCode: wrote E:\zhoulearn\kratos\verifyCode\cmd\verifyCode\wire_gen.go
'''
```
这里会生成一个 `wire_gen.go` 的一个文件, 这个文件帮助我们解决app依赖的作用(依赖注入)