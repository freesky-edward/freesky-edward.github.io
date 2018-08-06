---
layout: post
title: "基于元数据定义自动生成golang restful API"
date: 2018-06-13
type: tech
---

### go-swagger简介

go-swagger是一个swagger 2.0的golang实现，该项目能解析基于OpenAPI的swagger文档，并能基于这个swagger文档做如下工作：

1. 生成一个提供restful 的server框架
2. 生成一个调用该API的client
3. 生成使用文档

### 如何生成restful框架

有了这个工具项目，那么我们该如何使用呢，所有这些就将变得简单，大概需要如下几个步骤：

1. 使用swagger定义API
2. 生成API的metadata文档
3. 基于Metadata文档生成Server框架代码
4. 基于框架代码增加业务处理逻辑

首先第一步需要定义一个API，这里我推荐大家使用swagger的在线编辑器，地址：[https://editor2.swagger.io](https://editor2.swagger.io)

使用swagger的编辑工具定义API时使用的是标准的OpenAPI语法，具体的语法结构请参见[这里](https://github.com/OAI/OpenAPI-Specification/tree/master/versions), 目前推荐是用v2.0版本，当然更高版本也是没有问题的。

首先需要定义该API的常规元数据信息：

```
onsumes:
- application/json
info:
  description: this is api directory service
  title: The API dictory list
  version: 0.0.1
paths: {}
produces:
- application/json
schemes:
- http
swagger: "2.0"
```

在有了基础的描述以后再开始详细定义该API的结构，这个我们以定义一个API的字典服务为例，该服务只提供一个简单的服务API的查询功能。具体定义如下：

```
consumes:
- application/json
info:
  description: This is an API directory service 
    in a place it contains
  title: The API dictory list
  version: 0.0.1
paths: 
  /apis:
    get:
      tags:
        - metadaservices
      operationId: apis
      parameters:
      - name: keywords
        in: query
        type: string
      - name: start
        in: query
        type: integer
        format: int64
      - name: length
        in: query
        type: integer
        format: int64
        default: 100
      responses:
        200:
         description: List all APIs
         schema:
          $ref: "#/definitions/directory"
definitions:
  directory:
    type: object
    required:
      - version
    properties:
      version:
        type: string
        description: the directory version of this service
        minLength: 1
      services:
        type: array
        items:
          $ref: "#/definitions/service"
  service:
    type: object
    required:
     - name
     - version
     - restUrl
    properties:
      id:
        type: string
        description: the version of the specified service
      name:
        type: string
        description: the name of a specified service
        minLength: 1
      version:
        type: string
        description: the version of the specified service
      title:
        type: string
      restUrl:
        type: string
      description:
        type: string
      documentationLink:
        type: string
produces:
- application/json
schemes:
- http
swagger: "2.0"
```

这里只定义了一个资源——apis，该资源下只包含了一个get方法用于获取所有的api服务。

完成API的定义以后，可以通过工具导出该定义，具体通过点击左上方的[file] --> [download ...] 即可完成。

有了该API的定义文件后，接下来我们需要生成server框架代码。具体步骤如下：

1. 安装go-swagger, 安装方法有很多，这里我介绍基于源码编译

```
go get github.com/go-swagger/go-swagger/
export PATH=$PATH:$GOPATH/bin
```
这里关于golang的环境配置等就不再详细介绍，如果有不清楚出，请参见之前的[博客](../how-to-change-the-version-of-golang)

如果没有问题，那么系统就可以使用swagger命令

2. 使用go-swagger生成代码

```
swagger generate server -A Directory -f ./api-directory.yml -t /home/env/gopath/src/github.com/freesky-edward/directory/
```
-A 定义应用的名称
-f 指定API描述文档，即刚才导出的文档路径
-t 指定生成代码放置路径。

生成后的代码结构:

```
├── cmd
│   └── directory-server
│       └── main.go
├── models
│   ├── error.go
│   ├── directory.go
│   └── service.go
├── restapi
│   ├── configure_directory.go
│   ├── doc.go
│   ├── embedded_spec.go
│   ├── operations
│   │   ├── directory_api.go
│   │   └── metadataservies
│   │       ├── apis.go
│   │       ├── apis_parameters.go
│   │       ├── apis_responses.go
│   │       └── apis_urlbuilder.go
│   └── server.go
└── swagger.yml
```

所有directory相关的名字都是-A参数指定的，通过修改-A即可更改。

metadataservices可以在API定义文档里，get方法下指定的tags，里面文件名前缀apis，是指定的operationId，通过修改这些定义均可以修改。

有了这个框架后，接下来就是增加业务逻辑代码了，这里directory_api.go这个文件是可以修改，下次再生成代码时该文档是不会被覆盖的。

该文档主要是修改其中api handler逻辑。大致位置见下：

```
func  NewDirectoryAPI(spec  *loads.Document)  *DirectoryAPI  {

  return  &DirectoryAPI{

  handlers:  make(map[string]map[string]http.Handler),

  formats:  strfmt.Default,

  defaultConsumes:  "application/json",

  defaultProduces:  "application/json",

  customConsumers:  make(map[string]runtime.Consumer),

  customProducers:  make(map[string]runtime.Producer),

  ServerShutdown:  func()  {},

  spec:  spec,

  ServeError:  errors.ServeError,

  BasicAuthenticator:  security.BasicAuth,

  APIKeyAuthenticator:  security.APIKeyAuth,

  BearerAuthenticator:  security.BearerAuth,

  JSONConsumer:  runtime.JSONConsumer(),

  JSONProducer:  runtime.JSONProducer(),

  MetadaservicesApisHandler:  metadaservices.ApisHandlerFunc(func(params  metadaservices.ApisParams)  middleware.Responder  {

  return  middleware.NotImplemented("operation  MetadaservicesApis  has  not  yet  been  implemented")

  }),

  }

}
```

将其中的

```
return  middleware.NotImplemented("operation  MetadaservicesApis  has  not  yet  been  implemented")
```

修改为实际处理逻辑，所有的server代码就完成了。完成后就可以编译&运行该服务了

```
go install ./cmd/directory-server/
directory-server --help
```

### API变更

当需要变更API时，只需要修改swagger中API的定义，然后再重新生成一遍即可，其中的增加的业务逻辑文档是不会被覆盖的，这里有不再按步骤演示了。

需要注意的是除了业务逻辑之外的其它代码文件

