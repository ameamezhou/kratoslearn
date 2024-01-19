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

运行项目，使用 kratos 工具：

```shell
$ kratos run

DEBUG msg=config loaded: config.yaml format: yaml
INFO ts=2024-01-19T18:59:40+08:00 caller=http/server.go:317 service.id=DESKTOP-OFHN64T service.name= service.version= trace.id= span.id= msg=[HTTP] server listening on: [::]:8000
INFO ts=2024-01-19T18:59:40+08:00 caller=grpc/server.go:212 service.id=DESKTOP-OFHN64T service.name= service.version= trace.id= span.id= msg=[gRPC] server listening on: [::]:9000
```

以上信息，表示项目运行成功，正在监听 HTTP 8000, gRPC 9000 端口。

命令总结：

```shell
$ kratos new verifyCode
$ cd verifyCode
$ go mod tidy
$ go get github.com/google/wire/cmd/wire
$ go generate ./...
$ kratos run
```

## protobuf

原始版
```protobuf
syntax = "proto3";

package api.verifyCode;

option go_package = "verifyCode/api/verifyCode;verifyCode";
option java_multiple_files = true;
option java_package = "api.verifyCode";

service VerifyCode {
	rpc GetVerifyCode (GetVerifyCodeRequest) returns (GetVerifyCodeReply);
}


message GetVerifyCodeRequest {}
message GetVerifyCodeReply {}
```

完成版
```protobuf
syntax = "proto3";

package api.verifyCode;
// 生成的go代码所在的包
option go_package = "code/api/verifyCode;verifyCode";
// 定义 VerifyCode 服务
service VerifyCode {
    rpc GetVerifyCode (GetVerifyCodeRequest) returns (GetVerifyCodeReply);
}
// 类型常量
enum TYPE {
    DEFAULT = 0;
    DIGIT = 1;
    LETTER = 2;
    MIXED = 3;
};
// 定义 GetVerifyCodeRequest 消息
message GetVerifyCodeRequest {
    //    验证码长度
    uint32 length = 1;
    // 验证码类型
    TYPE type = 2;

}
// 定义 GetVerifyCodeReply 消息
message GetVerifyCodeReply {
    //    生成的验证码
    string code = 1;
}
```

### 基于 api/verifyCode/verifyCode.proto 文件生成 pb 代码和 grpc 代码

命令 `kratos proto client` 用于生成 client（Stub）相关代码

```shell
$ kratos proto client api/verifyCode/verifyCode.proto
```

代码会生成在 api/verifyCode/ 目录中：

```
# 类型定义代码
api/verifyCode/verifyCode.pb.go
# gRPC服务定义代码
api/verifyCode/verifyCode_grpc.pb.go

# 注意 http 代码只会在 proto 文件中声明了 http 时才会生成。我们这里不需要，因此没有生成，只存在以上两个文件
api/verifyCode/verifyCode_http.pb.go
```

注意，生成的以上代码不需要手动编辑。

### 基于 verifyCode.proto 文件生成 grpc 服务代码

命令 `kratos proto server` 用于生成服务相关代码：

```shell
$ kratos proto server api/verifyCode/verifyCode.proto -t internal/service
```

-t 选项指定生成文件所在位置

代码会生成在 internal/service 目录中：

```
verifycode.go
```

该文件定义了最基本的 VerifyCode 服务和对应的 GetVerifyCode 方法，其内容为：

internal/service/verifycode.go

```go
package service

import (
    "context"

    pb "verifyCode/api/verifyCode"
)

type VerifyCodeService struct {
    pb.UnimplementedVerifyCodeServer
}

func NewVerifyCodeService() *VerifyCodeService {
    return &VerifyCodeService{}
}

func (s *VerifyCodeService) GetVerifyCode(ctx context.Context, req *pb.GetVerifyCodeRequest) (*pb.GetVerifyCodeReply, error) {
    return &pb.GetVerifyCodeReply{}, nil
}
```

至此，通过编写 .proto 文件来生成接口基础代码的工作就完成了。

接下来还需要将生成的服务代码注册到 gRPC 服务中。


### 将生成的服务代码注册到 gRPC 服务中

**更新 internal/service/service.go 文件：**

```go
// ProviderSet is service providers.
var ProviderSet = wire.NewSet(NewGreeterService, NewVerifyCodeService)
```

在以上的 `wire.NewSet()` 调用中，添加第二个参数 NewVerifyCodeService。这个函数是用来生成 VerifyCodeService 服务的，定义在internal/service/verifycode.go 中。以上代码的意思就是告知 wire 依赖注入系统，如果需要 VerifyCodeService 的话，使用 NewVerifyCodeService 函数来构建。

**更新 internal/server/grpc.go 文件：**

在 NewGRPCServer 函数中：

1. 增加一个参数
2. 在函数体中，增加一行代码

用于将 VerifyCodeService 注册到 gRPC 服务中：

internal/server/grpc.go

```go
// 略 ...
// NewGRPCServer new a gRPC server.
func NewGRPCServer(c *conf.Server,
    greeter *service.GreeterService,
    // 增加下行，传递一个参数
    verifyCodeService *service.VerifyCodeService,
    logger log.Logger) *grpc.Server {
    // 略...
    srv := grpc.NewServer(opts...)
    v1.RegisterGreeterServer(srv, greeter)
    // 增加下行，将 VerifyCodeService 注册到 srv 中
    verifyCode.RegisterVerifyCodeServer(srv, VerifyCodeService)
}
```

### 生成依赖注入代码

运行：

```shell
$ go generate ./...
wire: code/cmd/code: wrote D:\apps\mashibing\laoma\backend\code\cmd\code\wire_gen.go
```

根据刚刚提供的 ProviderSet 来生成依赖注入的代码，保证项目正常运行。


