# Beego: Building Web API with Auto Generated API Document Support

## Introduction

We released [Beego 1.3](http://beego.me/docs/intro/releases.md#beego-1.3.0) last week with a very cool feature: Auto Generated API Document. This article will show you how to use this new feature.

## Why do we need Auto Generated API Document?

Now we need to build a API for the mobile app. We designed the API structures and wrote the API document prototype based on the requirements. We shared the document with the mobile developers to let them know how to use and test our API. However while developing the API structures will keep changing and we need to keep the API document updated. It's a pain for us. Many times after changing the API we forgot to update the API Document, it's also a pain for the user of the API. Another thing is how can we make the API be easy tested. It will be very good to have a interface where the user can test the API together with the document as well.

To walk through from this dilemma we found [Swagger](https://helloreverb.com/developers/swagger). You can see a [live demo](http://petstore.swagger.wordnik.com) here. 

>>> Swagger™ is a specification and complete framework implementation for describing, producing, consuming, and visualizing RESTful web services. 



With the auto generated document by integrating Swagger into beego we can have these benifits.

1. Standard declarative comments for API functions which makes 
2. The API code becomes more maintainable with the comments
3. The Swagger style API document will be generated automacally based on the comments.

## Creating the Beego API aplication

	go get -u github.com/beego/bee
	go get -u github.com/astaxie/beego
	
Go to `GOPATH/src` directory and execute command `bee api bapi` to create the API application. Then go to the application directory `cd bapi` and run `bee run -downdoc=true -docgen=true`.

This displays output similar to the following:

![](https://raw.githubusercontent.com/beego/beeblog/master/en-US/images/bee_api.png)

Loading up [http://127.0.0.1:8080/swagger/swagger-1/](http://127.0.0.1:8080/swagger/swagger-1/) in the browser shows the following output:

![](https://raw.githubusercontent.com/beego/beeblog/master/en-US/images/docs.png)

>>> You can only use 127.0.0.1 here other than loalhost to prevent the Ajax CORS issue.

Bang! Our live testable API document is ready to use. Now let's look into the details of the project we just created.


## Directory structure
You'll see the following directory structure created by `bee api`

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

- `main.go` Our bootstrap file.
- `bapi` The compiled binary file.
- `conf` The directory for configuration file.
- `controllers` Our controllers
- `models` Our models
- `docs` Our auto generated document
- `lastupdate.tmp` Cache file for `Annotation Router`
- `routers` Routers
- `swagger` A static directory for Swagger API related files.
- `test` The testcase for the application. You can run the testcases without run the application.

## The bootstrap main file

The following is the `main.go` file:

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

This is just a normal Beego bootstrap file with several lines of code to add `swagger` to static directory. 

Next let's see the router structure of our API application.

## Namespace Router

The router is using the `namespace` router. Two points two notice:
1. The API with auto generated document only support namespace router now.
2. It only support `NSNamespace` and `NSInclude` within two levels.

The following is our routers:

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
It created a `ns` variable with `beego.NewNamespace`. Actually `ns` variable is keeping a rouer tree and we can add this tree under any other router tree. This is a benifit of `namespace` that you can design the `namespace` in your module and add it to other modules with any prefix.

Let's take a look at `NewNamespace` function. Here is the definition:

    NewNamespace(prefix string, params ...innnerNamespace) *Namespace

the first parameter is the url prefix, the second parameter is the variable argument list with type `innnerNamespace`. 

The following is the definition of `innnerNamespace`:

	type innnerNamespace func(*Namespace)

It's a function that accept parameter `*Namespace`. In beego all the functions list below support return this function type `innnerNamespace`:

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

So we can use any function list above for the parameter list of `NewNamespace`.

Now let's take a look at our namespace router `ns`. It's a multi-level nested function. The first parameter is `/v1` which is the prefix of the router tree. The second and the third parameters are `beego.NSNamespace`. In the second level of `beego.NSNamespace`, we use `beego.NSInclude` to inject the `Annotation Router`. `beego.NSInclude` is designed for `Annotation Router`, we don't have any router settings here but only set up the controller. So how will this work? Let's go deeper into the `Annotation Router`.

## Annotation Router

`Annotation Router` are some comments set for Controller that Beego will parse them into the real routers automatically. Let's see what do we do to for it in `ObjectController` and `UserController`:

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

We can see there are bunch of comments above the functions. The last line is for the `Annotation Router` such as `//@router / [post]` which means routing `post` method for `/` to this function.

It has the exactlly same functionality as the router we were using befoer `beego.Router("/", &ObjectController{},"post:Post")` but in this case Beego will help you to register the router. But how does Beego do that? While the application starting, if the application is using `NSInclude` and the RunMode is `dev`, Beego will check if the router is already generated and if there is update, if true, Beego will use `AST` in Go to analyze the application's source code of the `controller` that `NSInclude` called and generate the routers and write to file `routers/commentsRouter.go`. 

>>> The router analysis will only happens in `dev` mode. So if you updated your code related to `Annotation Router`, you need to run `dev` mode at least once to generate the routers.

`Annotation Router` starts with `// @router` and accept two parameters: 1. The router URI 2. The supported methods. It need to be put above the router function. The order of the comments doesn't matter.

The first parameter router URI supports all the URI rules. Such as `/object/:key`, `/object` and `/cms_:id([0-9]+).html`.

The second parameter supported methods. Multiple methods separated by `,` such as `[post,get]`. But the auto generated API document only supports one method which means if there are multiple methods in the list, the auto generated API document will only take the first method. We don't recommend you to set multiple methods for one function.

By having these basic understanding of `Annotation Router` let's start the main topic today: Auto Generated API Document.

## Auto Generated API Document
Auto Generated API Document is the elegant swagger document generated by Beego based on the annotation comments automatically. The benifit of this approach is you only write the comments for the functions once and you get the live testable API document. They will always keep synchronized. It makes the source code maintainable as well as makes the API document testable. Two birds in one shot.

So how can we generate the API Document?

Here is how can we implement this approach:
1. Swagger's [schema](https://github.com/wordnik/swagger-spec/tree/master/schemas/v1.2) is a bunch of JSON data.
2. We write some annotations for the router functions.
3. Beego generate Swagger's schema based on the annotations.

The first step is describing the API:

### API Document
We mentioned it before in `router.go` there are bunch of annotaions. It's the description of this API application:

```
// @APIVersion 1.0.0
// @Title beego Test API
// @Description beego has a very cool tools to autogenerate documents for your API
// @Contact astaxie@gmail.com
// @TermsOfServiceUrl http://beego.me/
// @License Apache 2.0
// @LicenseUrl http://www.apache.org/licenses/LICENSE-2.0.html
```

Following are the flags we can use:

- @APIVersion
- @Title
- @Description
- @Contact
- @TermsOfServiceUrl
- @License
- @LicenseUrl

All of them are optional followed by a string. The following is the doc we generated: http://127.0.0.1:8080/docs:

```
{
  "apiVersion": "1.0.0",
  "swaggerVersion": "1.2",
  "apis": [
    {
      "path": "/object",
      "description": "Operations about object\n"
    },
    {
      "path": "/user",
      "description": "Operations about Users\n"
    }
  ],
  "info": {
    "title": "beego Test API",
    "description": "beego has a very cool tools to autogenerate documents for your API",
    "contact": "astaxie@gmail.com",
    "termsOfServiceUrl": "http://beego.me/",
    "license": "Url http://www.apache.org/licenses/LICENSE-2.0.html"
  }
}
```

This is the information fetched by the first request. How do we generate the `apis` object? Beego fetched it from your namespace source code. Right now it only supports two and only two level nested namespace. The first level is the baseUrl. The path comes from the second level namespace's prefix. So where does the description come from?

### Annotation for Controller
We add the annotation for controller and it's methods to describe the controller.

```
// Operations about object
type ObjectController struct {
	beego.Controller
}
``` 
The comment above describes the controller. The comment following for the methods will describe the router, parameters and the return value.

```
// @Title Get
// @Description find object by objectid
// @Param	objectId		path 	string	true		"the objectid you want to get"
// @Success 200 {object} models.Object
// @Failure 403 :objectId is empty
// @router /:objectId [get]
func (this *ObjectController) Get() {
}
```

Let's take a look at these annotations:

- @Title

    Title of the interface. Unique and optional.

    Format: followed by a string

- @Description

    Description of the interface. Unique and optional.
	
    Format: followed by a string
	
- @Param

    The request parameters for the interface. Multiple and optional.

    Format: `parameter name` `parameter sending type` `parameter data type` `required` `description comment`
	
    `parameter name` is a string

    `parameter sending type` can be any one of the following values:
    * `query` means the parameter in url send by GET. such as ?aa=bb&cc=dd
    * `form` means the parameter send by POST.
    * `path` means the parameter in the url path, such as for `/user/{uid}` uid is a parameter with `path` type.
    * `body` means the raw dada send from request body.
    * `header` means the parameter in request header

    `parameter data type` can be any one of the following values:
	* `string`
	* `int`
	* `int64`
	* `object path` The object path is related to the project path. For example `models.Object` means `Object` object under `models` directory.
	
    `description comment` is a string
	
    `required` is true or false
	

- @Success

    The success message returned to client. 

    Format: `status code` `return type` `return message or return object path`

    * `status code` is the standard HTTP status code, 200 201 etc
    * `return type` {object} is the object and will get the object by `return object path`. Other value will return `return message`.
    * `return object path` The object path is the relative path to the project path. For example `models.Object` means `Object` object under `models` directory.
 
- @Failure

    The failure message returned to client. 

    Format: `status code` `error message`
    
- @router

    The Annotation Router we discussed before.

>>> Every format above, parameters can be separated by on or more `'\t', '\n', '\v', '\f', '\r', ' ', U+0085 (NEL), U+00A0 (NBSP)`

Following code is the JSON data we generated from these annotations.

```
{
  "path": "/object/{objectId}",
  "description": "",
  "operations": [
    {
      "httpMethod": "GET",
      "nickname": "Get",
      "type": "",
      "summary": "find object by objectid",
      "parameters": [
        {
          "paramType": "path",
          "name": "objectId",
          "description": "\"the objectid you want to get\"",
          "dataType": "string",
          "type": "",
          "format": "",
          "allowMultiple": false,
          "required": true,
          "minimum": 0,
          "maximum": 0
        }
      ],
      "responseMessages": [
        {
          "code": 200,
          "message": "models.Object",
          "responseModel": "Object"
        },
        {
          "code": 403,
          "message": ":objectId is empty",
          "responseModel": ""
        }
      ]
    }
  ]
}
```	


### Customized Object Annotation
Following is the definition of our object:

```
type Object struct {
	ObjectId   string
	Score      int64
	PlayerName string
}
```

The auto generated object description for document is:

```
Object {
ObjectId (string, optional): ,
PlayerName (string, optional): ,
Score (int64, optional):
}
```

We can see all the fields are `optional`, we can change it by adding `tag` as showing below:

```
type Object struct {
	ObjectId   string	`required:"true" description:"object id"`
	Score      int64		`required:"true" description:"players's scores"`
	PlayerName string	`required:"true" description:"plaers name, used in system"`
}
```

To change the returned fields name, you can add `tag` with name json or thrife as showing below:

```
type Object struct {
	ObjectId   string	`json:"object_id"`
	Score      int64		`json:"player_score"`
	PlayerName string	`json:"player_name"`
}
```

The auto generated object description for document is:
```
Object {
object_id (string, optional): ,
player_score (string, optional): ,
player_name (int64, optional):
}
```

## FAQ

1. Q: bee doesn't have the command we ran befoer.

   A: Please upgrade bee to latest version.

1. Q: The document is generated, but I can't test the API.

   A: It might because of the CORS issue. please make sure the url and port you request is same as the application.

1. Q: Some other exceptions.

   A: Please report the issue to us. We will try to fix it ASAP.