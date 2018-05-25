# MVC

![](https://github.com/kataras/iris/raw/master/_examples/mvc/web_mvc_diagram.png)
Iris has **first-class support for the MVC (Model View Controller) pattern**, you'll not find
these stuff anywhere else in the Go world.

Iris web framework supports Request data, Models, Persistence Data and Binding
with the fastest possible execution.

**Characteristics**

All HTTP Methods are supported, for example if want to serve `GET`
then the controller should have a function named `Get()`,
you can define more than one method function to serve in the same Controller.

Serve custom controller's struct's methods as handlers with custom paths(even with regex parametermized path) via the `BeforeActivation` custom event callback, per-controller. Example:

```go
import (
    "github.com/kataras/iris"
    "github.com/kataras/iris/mvc"
)

func main() {
    app := iris.New()
    mvc.Configure(app.Party("/root"), myMVC)
    app.Run(iris.Addr(":8080"))
}

func myMVC(app *mvc.Application) {
    // app.Register(...)
    // app.Router.Use/UseGlobal/Done(...)
    app.Handle(new(MyController))
}

type MyController struct {}

func (m *MyController) BeforeActivation(b mvc.BeforeActivation) {
    // b.Dependencies().Add/Remove
    // b.Router().Use/UseGlobal/Done // and any standard API call you already know

    // 1-> Method
    // 2-> Path
    // 3-> The controller's function name to be parsed as handler
    // 4-> Any handlers that should run before the MyCustomHandler
    b.Handle("GET", "/something/{id:long}", "MyCustomHandler", anyMiddleware...)
}

// GET: http://localhost:8080/root
func (m *MyController) Get() string { return "Hey" }

// GET: http://localhost:8080/root/something/{id:long}
func (m *MyController) MyCustomHandler(id int64) string { return "MyCustomHandler says Hey" }
```

Persistence data inside your Controller struct (share data between requests)
by defining services to the Dependencies or have a `Singleton` controller scope.

Share the dependencies between controllers or register them on a parent MVC Application, and ability
to modify dependencies per-controller on the `BeforeActivation` optional event callback inside a Controller,
i.e `func(c *MyController) BeforeActivation(b mvc.BeforeActivation) { b.Dependencies().Add/Remove(...) }`.

Access to the `Context` as a controller's field(no manual binding is neede) i.e `Ctx iris.Context` or via a method's input argument, i.e `func(ctx iris.Context, otherArguments...)`.

Models inside your Controller struct (set-ed at the Method function and rendered by the View).
You can return models from a controller's method or set a field in the request lifecycle
and return that field to another method, in the same request lifecycle.

Flow as you used to, mvc application has its own `Router` which is a type of `iris/router.Party`, the standard iris api.
`Controllers` can be registered to any `Party`, including Subdomains, the Party's begin and done handlers work as expected.

Optional `BeginRequest(ctx)` function to perform any initialization before the method execution,
useful to call middlewares or when many methods use the same collection of data.

Optional `EndRequest(ctx)` function to perform any finalization after any method executed.

Inheritance, recursively, see for example our `mvc.SessionController`, it has the `Session *sessions.Session` and `Manager *sessions.Sessions` as embedded fields
which are filled by its `BeginRequest`, [here](https://github.com/kataras/iris/blob/master/mvc/session_controller.go).
This is just an example, you could use the `sessions.Session` which returned from the manager's `Start` as a dynamic dependency to the MVC Application, i.e
`mvcApp.Register(sessions.New(sessions.Config{Cookie: "iris_session_id"}).Start)`.

Access to the dynamic path parameters via the controller's methods' input arguments, no binding is needed.
When you use the Iris' default syntax to parse handlers from a controller, you need to suffix the methods
with the `By` word, uppercase is a new sub path. Example:

If `mvc.New(app.Party("/user")).Handle(new(user.Controller))`

- `func(*Controller) Get()` - `GET:/user`.
- `func(*Controller) Post()` - `POST:/user`.
- `func(*Controller) GetLogin()` - `GET:/user/login`
- `func(*Controller) PostLogin()` - `POST:/user/login`
- `func(*Controller) GetProfileFollowers()` - `GET:/user/profile/followers`
- `func(*Controller) PostProfileFollowers()` - `POST:/user/profile/followers`
- `func(*Controller) GetBy(id int64)` - `GET:/user/{param:long}`
- `func(*Controller) PostBy(id int64)` - `POST:/user/{param:long}`

If `mvc.New(app.Party("/profile")).Handle(new(profile.Controller))`

- `func(*Controller) GetBy(username string)` - `GET:/profile/{param:string}`

If `mvc.New(app.Party("/assets")).Handle(new(file.Controller))`

- `func(*Controller) GetByWildard(path string)` - `GET:/assets/{param:path}`

    Supported types for method functions receivers: int, int64, bool and string.

Response via output arguments, optionally, i.e

```go
func(c *ExampleController) Get() string |
                                (string, string) |
                                (string, int) |
                                int |
                                (int, string) |
                                (string, error) |
                                error |
                                (int, error) |
                                (any, bool) |
                                (customStruct, error) |
                                customStruct |
                                (customStruct, int) |
                                (customStruct, string) |
                                mvc.Result or (mvc.Result, error)
```

where [mvc.Result](https://github.com/kataras/iris/blob/master/mvc/go19.go#L10) is an [interface](https://github.com/kataras/iris/blob/master/hero/func_result.go#L18) which contains only that function: `Dispatch(ctx iris.Context)`.

## Using Iris MVC for code reuse

By creating components that are independent of one another, developers are able to reuse components quickly and easily in other applications. The same (or similar) view for one application can be refactored for another application with different data because the view is simply handling how the data is being displayed to the user.

If you're new to back-end web development read about the MVC architectural pattern first, a good start is that [wikipedia article](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller).

## Quick MVC Tutorial Part 1

This example is equivalent to the
https://github.com/kataras/iris/blob/master/_examples/hello-world/main.go

It seems that additional code you
have to write doesn't worth it
but remember that, this example
does not make use of iris mvc features like
the Model, Persistence or the View engine neither the Session,
it's very simple for learning purposes,
probably you'll never use such
as simple controller anywhere in your app.

The cost we have on this example for using MVC
on the "/hello" path which serves JSON
is ~2MB per 20MB throughput on my personal laptop,
it's tolerated for the majority of the applications
but you can choose
what suits you best with Iris, low-level handlers: performance
or high-level controllers: easier to maintain and smaller codebase on large applications.

```go
package main

import (
	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"

	"github.com/kataras/iris/middleware/logger"
	"github.com/kataras/iris/middleware/recover"
)

func main() {
	app := iris.New()
	// Optionally, add two built'n handlers
	// that can recover from any http-relative panics
	// and log the requests to the terminal.
	app.Use(recover.New())
	app.Use(logger.New())

	// Serve a controller based on the root Router, "/".
	mvc.New(app).Handle(new(ExampleController))

	// http://localhost:8080
	// http://localhost:8080/ping
	// http://localhost:8080/hello
	// http://localhost:8080/custom_path
	app.Run(iris.Addr(":8080"))
}

// ExampleController serves the "/", "/ping" and "/hello".
type ExampleController struct{}

// Get serves
// Method:   GET
// Resource: http://localhost:8080
func (c *ExampleController) Get() mvc.Result {
	return mvc.Response{
		ContentType: "text/html",
		Text:        "<h1>Welcome</h1>",
	}
}

// GetPing serves
// Method:   GET
// Resource: http://localhost:8080/ping
func (c *ExampleController) GetPing() string {
	return "pong"
}

// GetHello serves
// Method:   GET
// Resource: http://localhost:8080/hello
func (c *ExampleController) GetHello() interface{} {
	return map[string]string{"message": "Hello Iris!"}
}

// BeforeActivation called once, before the controller adapted to the main application
// and of course before the server ran.
// After version 9 you can also add custom routes for a specific controller's methods.
// Here you can register custom method's handlers
// use the standard router with `ca.Router` to do something that you can do without mvc as well,
// and add dependencies that will be binded to a controller's fields or method function's input arguments.
func (c *ExampleController) BeforeActivation(b mvc.BeforeActivation) {
	anyMiddlewareHere := func(ctx iris.Context) {
		ctx.Application().Logger().Warnf("Inside /custom_path")
		ctx.Next()
	}
	b.Handle("GET", "/custom_path", "CustomHandlerWithoutFollowingTheNamingGuide", anyMiddlewareHere)

	// or even add a global middleware based on this controller's router,
	// which in this example is the root "/":
	// b.Router().Use(myMiddleware)
}

// CustomHandlerWithoutFollowingTheNamingGuide serves
// Method:   GET
// Resource: http://localhost:8080/custom_path
func (c *ExampleController) CustomHandlerWithoutFollowingTheNamingGuide() string {
	return "hello from the custom handler without following the naming guide"
}

// GetUserBy serves
// Method:   GET
// Resource: http://localhost:8080/user/{username:string}
// By is a reserved "keyword" to tell the framework that you're going to
// bind path parameters in the function's input arguments, and it also
// helps to have "Get" and "GetBy" in the same controller.
//
// func (c *ExampleController) GetUserBy(username string) mvc.Result {
// 	return mvc.View{
// 		Name: "user/username.html",
// 		Data: username,
// 	}
// }

/* Can use more than one, the factory will make sure
that the correct http methods are being registered for each route
for this controller, uncomment these if you want:

func (c *ExampleController) Post() {}
func (c *ExampleController) Put() {}
func (c *ExampleController) Delete() {}
func (c *ExampleController) Connect() {}
func (c *ExampleController) Head() {}
func (c *ExampleController) Patch() {}
func (c *ExampleController) Options() {}
func (c *ExampleController) Trace() {}
*/

/*
func (c *ExampleController) All() {}
//        OR
func (c *ExampleController) Any() {}



func (c *ExampleController) BeforeActivation(b mvc.BeforeActivation) {
	// 1 -> the HTTP Method
	// 2 -> the route's path
	// 3 -> this controller's method name that should be handler for that route.
	b.Handle("GET", "/mypath/{param}", "DoIt", optionalMiddlewareHere...)
}

// After activation, all dependencies are set-ed - so read only access on them
// but still possible to add custom controller or simple standard handlers.
func (c *ExampleController) AfterActivation(a mvc.AfterActivation) {}

*/
```

> The [_examples/mvc](https://github.com/kataras/iris/tree/master/_examples/mvc) and [mvc/controller_test.go](https://github.com/kataras/iris/blob/master/mvc/controller_test.go) files explain each feature with simple paradigms, they show how you can take advandage of the Iris MVC Binder, Iris MVC Models and many more...

Every `exported` func prefixed with an HTTP Method(`Get`, `Post`, `Put`, `Delete`...) in a controller is callable as an HTTP endpoint. In the sample above, all funcs writes a string to the response. Note the comments preceding each method.

An HTTP endpoint is a targetable URL in the web application, such as `http://localhost:8080/helloworld`, and combines the protocol used: HTTP, the network location of the web server (including the TCP port): `localhost:8080` and the target URI `/helloworld`.

The first comment states this is an [HTTP GET](https://www.w3schools.com/tags/ref_httpmethods.asp) method that is invoked by appending "/helloworld" to the base URL. The third comment specifies an [HTTP GET](https://www.w3schools.com/tags/ref_httpmethods.asp) method that is invoked by appending "/helloworld/welcome" to the URL.

Controller knows how to handle the "name" on `GetBy` or the "name" and "numTimes" at `GetWelcomeBy`, because of the `By` keyword, and builds the dynamic route without boilerplate; the third comment specifies an [HTTP GET](https://www.w3schools.com/tags/ref_httpmethods.asp) dynamic method that is invoked by any URL that starts with "/helloworld/welcome" and followed by two more path parts, the first one can accept any value and the second can accept only numbers, i,e: "http://localhost:8080/helloworld/welcome/golang/32719", otherwise a [404 Not Found HTTP Error](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.5) will be sent to the client instead.

----

Click [here](mvc_2.md) to navigate to the `Movies MVC Application` sub section.

----