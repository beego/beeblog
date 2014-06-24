# beego API开发以及自动化文档
beego1.3版本已经在上个星期发布了，但是还是有很多人不了解如何来进行开发，也是在一步一步的测试中开发，期间QQ群里面很多人都问我如何开发，我的业余时间实在是排的太满了，实在是没办法一一回复大家，在这里和大家说声对不起，这两天我又不断的改进，写了一个应用示例展示如何使用beego开发API已经自动化文档和测试，这里就和大家详细的解说一下。

## 自动化文档开发的初衷
我们需要开发一个API应用，然后需要和手机组的开发人员一起合作，当然我们首先想到的是文档先行，我们也根据之前的经验写了我们需要的API原型文档，我们还是根据github的文档格式写了一些漂亮的文档，但是我们开始担心这个文档如果两边不同步怎么办？因为毕竟是原型文档，变动是必不可少的。手机组有一个同事之前在雅虎工作过，他推荐我看一个swagger的应用，看了swagger的标准和文档化的要求，感觉太棒了，这个简直就是神器啊，通过swagger可以方便的查看API的文档，同时使用API的用户可以直接通过swagger进行请求和获取结果。所以我就开始学习swagger的标准，同时开始进行Go源码的研究，通过Go里面的AST进行源码分析，针对comments解析，然后生成swagger标准的json格式，这样最后就可以和swagger完美结合了。

这样做的好处有三个：

1. 注释标准化
2. 有了注释之后，以后API代码维护相当方便
3. 根据注释自动化生成文档，方便调用的用户查看和测试

## beego API应用入门
请大家更新到最新的bee和beego

	go get -u github.com/beego/bee
	go get -u github.com/astaxie/beego
	
然后进入到你的`GOPATH/src`目录，执行命令`bee api bapi`,进入目录`cd bapi`,执行命令`bee run -downdoc=true -docgen=true`.请看下面我执行的效果图：

![](https://raw.githubusercontent.com/beego/beeblog/master/zh-CN/images/bee_api.png)

执行完成之后就打开浏览器，输入URL:[http://127.0.0.1:8080/swagger/swagger-1/](http://127.0.0.1:8080/swagger/swagger-1/)

>>> 记住这里必须要用127.0.0.1，不能使用localhost，存在CORS问题，Ajax跨域

![](https://raw.githubusercontent.com/beego/beeblog/master/zh-CN/images/docs.png)

我们的效果和应用都出来了，很酷很炫吧，那这后面到底采用了怎么样的一些技术呢？让我们一步一步来讲解这些细节：

## 项目目录
我们首先来了解一下`bee api`创建的应用的目录结构：

```
|-- bapi
|-- conf
|   `-- app.conf
|-- controllers
|   |-- object.go
|   `-- user.go
|-- docs
|   |-- doc.go
|   `-- docs.go
|-- lastupdate.tmp
|-- main.go
|-- models
|   |-- object.go
|   `-- user.go
|-- routers
|   |-- commentsRouter.go
|   `-- router.go
|-- swagger
`-- tests
    `-- default_test.go
```

- main.go 是程序的统一入口文件
- bapi 是生成的二进制文件
- conf 配置文件目录，app.conf
- controllers 控制器目录，主要是逻辑的处理
- models 是数据处理层的目录
- docs 是自动化生成文档的目录
- lastupdate.tmp 是一个注解路由的缓存文件
- routers是路由目录，主要涉及一些路由规则
- swagger 是一个html静态资源目录，是通过bee自动下载的，主要就是展示我们看到的界面及测试
- test 目录是针对应用的测试用例，beego相比其他revel框架的好处之一就是无需启动应用就可以执行test case。

## 入口文件main
我们第一步先来看一下入口是怎么写的？

```
package main

import (
	_ "bapi/docs"
	_ "bapi/routers"

	"github.com/astaxie/beego"
)

func main() {
	if beego.RunMode == "dev" {
		beego.DirectoryIndex = true
		beego.StaticDir["/swagger"] = "swagger"
	}
	beego.Run()
}
```

入口文件就是一个普通的beego应用的标准代码，只是这里多了几行代码，把swagger加入了static，因为我们需要把文档服务器集成到beego的API应用中来。
## namespace路由


## 注解路由

## 自动化文档

### API文档

### 子路由文档

## 常见错误及问题

