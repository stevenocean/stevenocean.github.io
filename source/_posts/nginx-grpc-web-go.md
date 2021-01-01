title: 使用 grpc-web, vue 和 Nginx 搭建一个简单 todo 示例
tags:
  - grpc
  - grpc-web
  - go
categories:
  - grpc
  - grpc-web
  - Nginx
author: Steven Hu
date: 2020-06-20 15:03:00

----

# 关于 gRPC

[gRPC](https://www.grpc.io/) 是一个高性能、通用的开源 RPC 框架，其由 Google 主要面向移动应用开发并基于 HTTP/2 协议标准而设计，基于 [ProtoBuf (Protocol Buffers)](https://developers.google.com/protocol-buffers/) 序列化协议开发，且支持众多开发语言（）。

gRPC 提供了一种简单的方法来精确地定义服务和为iOS、Android 和 后台支持服务自动生成可靠性很强的客户端功能库。客户端充分利用高级流和链接功能，从而有助于节省带宽、降低的 TCP 链接次数、节省 CPU 使用、和电池寿命。下图为 gRPC 结构图：

<!-- more -->

{% asset_img 2020-06-19_22-40-38.jpg %}

gRPC 具有如下一些重要的特性：

- gRPC 默认通过 Protocol Buffers 来定义接口，可以制定更加严格规范的接口约束；
- 而基于 ProtoBuf 可以将数据序列化为二进制格式，从而大幅度减少数据量，进而大幅度的提升性能；
- 支持流式通信（Streaming），基于 HTTP/2 协议传输可以实现 Streaming 功能模式，可提供更快的响应和更高的性能；
- 支持多种语言，包括：Android Java、C++、C#/.NET、Dart、Go、Python、Web 等等；

# 关于 grpc-web

[gRPC-web](https://github.com/grpc/grpc-web) 是针对浏览器端的 gRPC 的 Javascript 实现，是需要配合代理来一起使用的。

在官网上的示例都是使用 Envoy 作为代理，本文主要采用 Nginx 作为代理，从 nginx-1.13.10 引入了对 grpc 的支持（grpc_pass），可以代理 gRPC TCP 连接，还可以终止、检查和跟踪 gRPC 的方法调用，可以如下：

- 发布 gRPC 服务，然后使用 NGINX 应用 HTTP/2 TLS 加密、限速、基于 IP 的访问控制列表和日志记录；也可以使用未加密的 HTTP/2（h2c cleartext）或者在服务之上封装 TLS 加密和认证信息；
- 通过单个端点发布多个 gRPC 服务，使用 NGINX 检查并跟踪每个内部服务的调用；
- 使用 Round Robin, Least Connections 或其他方法在集群分配调用，对 gRPC 服务集群进行负载均衡；

下图为一种简单的使用 Nginx 暴露 gRPC 服务的方案：

{% asset_img gRPC-nginx-proxy.png %}

在客户端和服务端之间的 Nginx 可以为服务端的应用提供一个稳定可靠的网关。

下面通过一个简单完整的 todo 示例来演示 gRPC-web + Nginx + Go 服务端的应用。

# 创建 todo.proto 协议文件

定义接口和数据类型：

```protobuf
syntax = "proto3";
package todo;

message getTodoParams{}

message addTodoParams {
  string task = 1;
}

message deleteTodoParams {
  string id = 1;
}

message todoObject {
  string id = 1;
  string task = 2;
}

message todoResponse {
  repeated todoObject todos = 1;
}

message deleteResponse {
  string message = 1;
}  

service todoService {
  rpc addTodo(addTodoParams) returns (todoObject) {}
  rpc deleteTodo(deleteTodoParams) returns (deleteResponse) {}
  rpc getTodos(getTodoParams) returns (todoResponse) {}
}
```

这里使用的是 protobuf 的 v3 版本规范。通过 service 定义了一个 todoService，其中包含了 addTodo、deleteTodo、getTodos 等是三个 RPC 方法，以及通过 message 定义这些方法中请求参数和返回类型。

> [详细的 protobuf 规范参考](https://developers.google.com/protocol-buffers/docs/proto3)

# 服务端代码生成

本示例的服务端采用 Go，下面使用 protoc 编译生成工具来生成服务端代码，执行如下命令行：

```shell
➜  todo protoc -I ./todo/ --go_out=plugins=grpc:./todo/ ./todo/todo.proto
➜  todo tree
.
├── todo
│   ├── todo.pb.go
│   └── todo.proto
└── todo-client
```

在上面的 protoc 命令中，指定 proto 文件的路径在 ./todo/ 下面，且指定了生成的源码文件 todo.pb.go 输出路径也为 ./todo/。

下面我们创建一个 handler.go 来处理这些 RPC 方法：

```go
package todo

import (
  "log"
  "golang.org/x/net/context"
  "github.com/satori/go.uuid"
)

type Server struct {
  Todos []*TodoObject
}

func (s *Server) AddTodo(ctx context.Context, newTodo *AddTodoParams) (*TodoObject, error) {
  log.Printf("Received new task %s", newTodo.Task)
  todoObject := &TodoObject{
    Id: uuid.NewV1().String(),
    Task: newTodo.Task,
  }
  s.Todos = append(s.Todos, todoObject)
  return todoObject, nil
}

func (s *Server) GetTodos(ctx context.Context, _ *GetTodoParams) (*TodoResponse, error) {
  log.Printf("get tasks")
  return &TodoResponse{Todos: s.Todos}, nil
}

func (s *Server) DeleteTodo(ctx context.Context, delTodo *DeleteTodoParams) (*DeleteResponse, error) {
  var updatedTodos []*TodoObject
  for index, todo := range s.Todos {
    if(todo.Id == delTodo.Id) {
      updatedTodos = append(s.Todos[:index], s.Todos[index + 1:]...)
        break;
    }
  }
  s.Todos = updatedTodos
  return &DeleteResponse{Message: "success"}, nil
}
```

如上示例，在服务运行时的 Todos 数组中存储所有 todo 记录，所有的 RPC 方法都 “绑定” 在 Server 结构体下，其中：

- **AddTodo**：其中使用了 uuid 包来为每一条 todo 记录生成唯一的标识，并被 append 到 Todos 数组后；
- **DeleteTodo**：根据 todo.id 找到对应的 todo 记录并从 Todos 列表中移除和更新；
- **GetTodo**：简单的返回整个 Todo 列表；

> [Go support for Protocol Buffers](https://github.com/golang/protobuf)

# 服务端入口程序

我们还需要启动 RPC 服务器的入口程序 server.go，位于 todo 根目录下，此时目录结构如下：

```shell
➜  todo tree
.
├── server.go
├── todo
│   ├── handler.go
│   ├── todo.pb.go
│   └── todo.proto
└── todo-client
```

server.go 的源码如下：

```go
package main

import (
    "fmt"
    "log"
    "net"

    "github.com/thearavind/grpc-todo/todo"
    "google.golang.org/grpc"
)

func main() {
    lis, err: = net.Listen("tcp", fmt.Sprintf(":%d", 50096))
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    s: = todo.Server {}
    grpcServer: = grpc.NewServer()
    // attach the todo service to the server
    todo.RegisterTodoServiceServer(grpcServer, & s)

    if err: = grpcServer.Serve(lis);
    err != nil {
        log.Fatalf("failed to serve: %s", err)
    } else {
        log.Printf("Server started successfully")
    }
}
```

在 main() 主函数入口中，首先启动 TCP 监听，端口为 50096，之后创建 Server 和 grpcServer 实例，并通过 RegisterTodoServiceServer 将我们的 todo 服务注册到这个新建的 grpcServer 上，最后启动 grpc server。

我们可以通过 go run server.go 来启动服务端：

# Nginx 代理配置

Nginx 配置如下：

```nginx
server {
    listen       9080;
    server_name  localhost;

    location / {
        grpc_set_header     Content-Type application/grpc;
        grpc_pass           grpc://localhost:50098;

        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' "*";
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, DELETE';
            add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Transfer-Encoding,Custom-Header-1,X-Accept-Content-Transfer-Encoding,X-Accept-Response-Streaming,X-User-Agent,X-Grpc-Web';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        if ($request_method = 'POST') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Transfer-Encoding,Custom-Header-1,X-Accept-Content-Transfer-Encoding,X-Accept-Response-Streaming,X-User-Agent,X-Grpc-Web';
            add_header 'Access-Control-Expose-Headers' 'Content-Transfer-Encoding,Grpc-Message,Grpc-Status';
        }
    }
}
```

通过 grpc_pass 来实现 grpc 协议的代理，重定向至服务端 50098 端口。

注意：这里需要通过 grpc_set_header 来设置 Content-Type 为 application/grpc，默认为 application/grpc-web。

# 使用 vuejs 实现的前端应用

我们通过 vue 来创建一个简单的 todo-client：

```shell
➜  todo vue create todo-client
➜  todo tree -L 2 
.
├── go.mod
├── go.sum
├── server.go
├── todo
│   ├── handler.go
│   ├── todo.pb.go
│   └── todo.proto
└── todo-client
    ├── babel.config.js
    ├── node_modules
    ├── package.json
    ├── package-lock.json
    ├── public
    ├── README.md
    └── src
```

下面通过 grpc-web 插件生成 Javascript 的接口代码：

```bash
➜  todo protoc -I=./todo todo.proto --js_out=import_style=commonjs:./todo-client/src --grpc-web_out=import_style=commonjs,mode=grpcweb:./todo-client/src
➜  todo tree -L 2
.
├── go.mod
├── go.sum
├── server.go
├── todo
│   ├── handler.go
│   ├── todo.pb.go
│   └── todo.proto
└── todo-client
    ├── babel.config.js
    ├── node_modules
    ├── package.json
    ├── package-lock.json
    ├── public
    ├── README.md
    └── src
        ├── App.vue
        ├── assets
        ├── components
        ├── main.js
        ├── todo_grpc_web_pb.js
        └── todo_pb.js
```

在 todo-client/src 目录下生成 todo_grpc_web_pb.js 和 todo_pb.js 文件，下面我们在 App.vue 中引入：

```javascript
<script>
import { addTodoParams, getTodoParams, deleteTodoParams } from "./todo_pb";
import { todoServiceClient } from "./todo_grpc_web_pb";

export default {
  name: 'App',
  components: {
  },
  data: function() {
    return {
      inputField: "",
      todos: []
    };
  },
  created: function() {
    this.client = new todoServiceClient("http://192.168.1.188:9080", null, null);
    this.getTodos();
  },
  methods: {
    getTodos: function() {
      let getRequest = new getTodoParams();
      this.client.getTodos(getRequest, {}, (err, response) => {
        this.todos = response.toObject().todosList;
        console.log(this.todos);
      });
    },
    addTodo: function() {
      let request = new addTodoParams();
      request.setTask(this.inputField);
      this.client.addTodo(request, {}, () => {
        this.inputField = "";
        this.getTodos();
      });
    },
    deleteTodo: function(todo) {
      let deleteRequest = new deleteTodoParams();
      deleteRequest.setId(todo.id);
      this.client.deleteTodo(deleteRequest, {}, (err, response) => {
        if (response.getMessage() === "success") {
          this.getTodos();
        }
      });
      console.log("todo -> ", todo.id);
    }
  }
}
</script>
```

在 created 阶段创建 todoServiceClient 实例，其中第一个参数为代理的主机地址端口，这里为 Nginx 的（HTTP）服务地址和端口。在 getTodos, addTodo, deleteTodo 等方法中可以创建 RPC 访问的请求参数实例，并且在异步调用后，通过 response 来获取相应信息。如下为示例图：

{% asset_img 2020-06-20_17-16-36.jpg %}

# References

- [gRPC 官网](https://www.grpc.io/)
- [Protocol Buffer 官网](https://developers.google.com/protocol-buffers/)
- [gRPC over HTTP/2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
- [gRPC-web 官网](https://github.com/grpc/grpc-web)
- [Introducing gRPC Support with NGINX 1.13.10](https://www.nginx.com/blog/nginx-1-13-10-grpc/)
