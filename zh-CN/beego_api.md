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

入口文件就是一个普通的beego应用的标准代码，只是这里多了几行代码，把swagger加入了static，因为我们需要把文档服务器集成到beego的API应用中来。然后增加了docs的初始化引入，和router的效果一样。接下里我们先来看看自动化API的路由是怎么设计的
## namespace路由
自动化路由才有了namespace来进行设计，而且注意两点，第一目前只支持namespace的路由支持自动化文档，第二只支持NSNamespace和NSInclude解析，而且是只能两个层级，先看我们的路由设置：

```
func init() {
	ns := beego.NewNamespace("/v1",
		beego.NSNamespace("/object",
			beego.NSInclude(
				&controllers.ObjectController{},
			),
		),
		beego.NSNamespace("/user",
			beego.NSInclude(
				&controllers.UserController{},
			),
		),
	)
	beego.AddNamespace(ns)
}
```
我们先来看一下这个代码，首先是使用beego.NewNamespace创建一个ns的变量，这个变量里面其实就是存储了一棵路由树，我们可以把这棵树加到其他任意已经存在的树中去，这也就是namespace的好处，可以在任意的模块中设计自己的namespace，然后把这个namespace加到其他的应用中去，可以增加任意的前缀等。

这里我们分析一下NewNamespace这个函数，这个函数的定义是这样的`NewNamespace(prefix string, params ...innnerNamespace) *Namespace`，他的第一个参数就是前缀，第二个参数是`innnerNamespace`多参数，那么我们来看看`innnerNamespace`的定义是什么：

	type innnerNamespace func(*Namespace)

它是一个函数，也就是只要是符合参数是`*Namespace`的函数都可以。那么在beego里面定义了如下的方法支持返回这个函数类型：

- NSCond(cond namespaceCond) innnerNamespace
- NSBefore(filiterList ...FilterFunc) innnerNamespace
- NSAfter(filiterList ...FilterFunc) innnerNamespace
- NSInclude(cList …ControllerInterface) innnerNamespace
- NSRouter(rootpath string, c ControllerInterface, mappingMethods …string) innnerNamespace
- NSGet(rootpath string, f FilterFunc) innnerNamespace
- NSPost(rootpath string, f FilterFunc) innnerNamespace
- NSDelete(rootpath string, f FilterFunc) innnerNamespace
- NSPut(rootpath string, f FilterFunc) innnerNamespace
- NSHead(rootpath string, f FilterFunc) innnerNamespace
- NSOptions(rootpath string, f FilterFunc) innnerNamespace
- NSPatch(rootpath string, f FilterFunc) innnerNamespace
- NSAny(rootpath string, f FilterFunc) innnerNamespace
- NSHandler(rootpath string, h http.Handler) innnerNamespace
- NSAutoRouter(c ControllerInterface) innnerNamespace 
- NSAutoPrefix(prefix string, c ControllerInterface) innnerNamespace
- NSNamespace(prefix string, params …innnerNamespace) innnerNamespace

因此我们可以在`NewNamespace`这个函数的第二个参数列表中使用上面的任意函数作为参数调用。

我们看一下路由代码，这是一个层级嵌套的函数，第一个参数是`/v1`，即为`/v1`开头的路由树，第二个参数是`beego.NSNamespace`，第三个参数也是`beego.NSNamespace`，也就是路由树嵌套了路由树，而我们的`beego.NSNamespace`里面也是和`NewNamespace`一样的参数，第一个参数是路由前缀，第二个参数是slice参数。这里我们调用了`beego.NSInclude`来进行注解路由的引入，这个函数是专门为注解路由设计的，我们可以看到这个设计里面我们没有任何的路由信息，只是设置了前缀，那么这个的路由是在哪里设置的呢？我们接下来分析什么是注解路由。

## 注解路由
可能有些同学不了解什么是注解路由，也就是在Controller类上添加一个注释让框架给自动添加Route，那么我们来看一下`ObjectController`和`UserController`中怎么写路由注解的：

```
// Operations about object
type ObjectController struct {
	beego.Controller
}

// @Title create
// @Description create object
// @Param	body		body 	models.Object	true		"The object content"
// @Success 200 {string} models.Object.Id
// @Failure 403 body is empty
// @router / [post]
func (this *ObjectController) Post() {
	var ob models.Object
	json.Unmarshal(this.Ctx.Input.RequestBody, &ob)
	objectid := models.AddOne(ob)
	this.Data["json"] = map[string]string{"ObjectId": objectid}
	this.ServeJson()
}
```

我们看到我们的每一个函数上面有大段的注释，注解路由

## 自动化文档

### API文档

### 子路由文档

## 常见错误及问题

