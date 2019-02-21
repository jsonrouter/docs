# Contents
<div id="contents">
<h2>Main</h2>
<a href="#features">Features</a><br>
<a href="#httpRequest">http.Request request context</a><br>
<a href="#fasthttp">Fasthttp implementation example</a><br>
<h2>How it works</h2>
<a href="#howItWorks">How it works</a><br>
<a href="#classicStyle">Classic style</a><br>
<a href="#headersAndCORS">Headers & CORS setup</a><br>
<h2>Validation</h2>
<a href="#validation">Validation</a><br>
<a href="#JSONParam">JSON parameter validation</a><br>
<a href="#PathParam">Path parameter validation</a><br>
<h2>Other</h2>
<a href="#servingFiles">Serving files and folders</a><br>
<a href="#abstraction">Abstraction modules</a><br>
<a href="#autoGen">Auto-generated documentation</a>	<br>

</div>

#  

# Golang/Go HTTP router for websites & JSON APIs

Created by Alex Breadman, this software is released with a M.I.T. license.

UPDATES

Everything is now focussed around writing valid openAPI v1/v2 spec for you.

Initialising JSONrouter is done via the /platforms/* repos as a Router object.

<div id="features">

## Features

<a href="#contents">Top</a>

Node-based routing with no annoying conflicts when designing APIs.

Validation & sanitation of all variables passed as route params.

Validation & sanitation of request payload & all variables passed as JSON params.


Automatically creates documentaion for all endpoints that have a handler.

Automatically handles all client errors in a dependable way.

Allows custom validation & middleware functions.

Allows very complex routing without creating a mess.
</div>
<div id="httpRequest">

## http.Request request context

<a href="#contents">Top</a>

This package uses an interface to allow usage of different http implementations such as net/http package or valyala/fasthttp package. Unfortunately, fasthttp doesn't provide API identical to net/http, so a common interface type has been created to allow JSONrouter to operate with either dependency.

If you are using net/http the request context (instance of RequestInterface) allows access to the http.ResponseWriter & *http.Request via the .Res() and .R() functions respectively.

</div>
<div id="fasthttp">

## go - fasthttp implementation example

<a href="#contents">Top</a>

This version uses the fasthttp package rather than net/http package for http interfacing.

More information on fasthttp can be found here: https://github.com/valyala/fasthttp

```
package main

import (
	"net/http"
	//
	"github.com/jsonrouter/core/http"
	"github.com/jsonrouter/core/openapi/v2"
	"github.com/jsonrouter/core/validation"
	"github.com/jsonrouter/logging"
	"github.com/jsonrouter/logging/testing"
	"github.com/jsonrouter/platforms/standard"
)

type App struct {
	logger logging.Logger
}

type Product struct {
	name string
}

func main() {

	app := &App{
		logger: logs.NewClient().NewLogger("Server"),
	}

	router, err := jsonrouter.New(app.logger, openapiv2.New("localhost", "test"))
	if err != nil {
		panic(err)
	}

	api := router.Add("/api")

	api.Add("/product").Param(validation.StringExact(30), "productID").GET(
		app.api_product_get,
	).Description(
		"Gets the specified product",
	).Response(
		Product{},
	)

	http.ListenAndServe(":80", router)
}


func (app *App) api_product_get(req http.Request) *http.Status {
	// do something
	return nil
}

```

</div>
<div id="howItWorks">

# How it works

<a href="#contents">Top</a>

The root node (type *Node) which is returned when calling .NewRouter(...) represents the root path. Methods can be called on *Node type such as .Add(...) and .Param(...).

Constructing http path routes with GF is as simple as chaining these methods, or creating variables which represent endpoints.

</div>
<div id="classicStyle">

## Classic style

<a href="#contents">Top</a>

```

func apiUserPut(req http.Request, arg interface{}) *http.Status {

	id := req.Param("id").(string)
	firstName := req.BodyParam("firstName").(string)
	lastName := req.BodyParam("lastName").(string)

	userData := map[string]interface{}{
		"id":		id,
		"firstName":	strings.Title(firstName),
		"lastName:	strings.Title(lastName),
	}

	ok, user := db.Create("user", id, userData); if !ok { return req.Fail() }

	return req.Respond(user)
}

func apiUserGet(req http.Request, arg interface{}) *http.Status {

	id := req.Param("id").(string)

	ok, user := db.Get("user", id); if !ok { return req.Fail() }

	return req.Respond(user)
}

func apiUserPatch(req http.Request, arg interface{}) *http.Status {

	id := req.Param("id").(string)
	field := req.BodyParam("field").(string)
	value := req.BodyParam("value").(string)

	switch field {

		case "firstName":
		case "lastName":
		default:

			return req.Respond(400, "INVALID FIELD: "+field)

	}

	userData := map[string]interface{}{}
	userData[field] = value

	ok, user := db.Update("user", id, userData); if !ok { return req.Fail() }

	return req.Respond(user)
}

func apiUserDelete(req http.Request, arg interface{}) *http.Status {

	id := req.Param("id").(string)

	ok, user := db.Delete("user", id)
	if !ok {
		return req.Fail()
	}

	return nil
}

func main() {

	root, router := standard.NewRouter('www')

	v1 := root.Add("v1")

		user := v1.Add("user").Param(validation.String(64), "id")

			user.GET(
				app.apiUserGet,
			).Description(
				"Gets the user.",
			).Response(
				User{},
			)

			user.PUT(
				app.apiUserPut,
			).Description(
				"Creates new user.",
			).Required(
				&validation.Payload{
					"firstName": validation.String(2, 30),
					"lastName":	validation.String(2, 30),
				},
			).Response(
				User{},
			)

			user.DELETE(
				app.apiUserDelete,
			).Description(
				"Deletes the user.",
			)

			user.PATCH(
				app.apiUserPatch,
			).Description(
				"Patches user fields.",
			).Required(
				&validation.Payload{
					"field": validation.String(2, 12),
					"value": validation.String(2, 30),
				},
			).Response(
				User{},
			)

}

```

The above psuedocode would create these routes:

### PUT:

/v1/user/:id

### GET:

/v1/user/:id

### PATCH:

/v1/user/:id

### DELETE:

/v1/user/:id

As you can see you can create restful API structures, whilst having all sanitation and validation handled by a dependable package.

</div>
<div id="headersAndCORS">

# Headers & CORS setup

<a href="#contents">Top</a>

Currently any headers settings applied to the node will apply to all children.

```

	v1 := rootPath.Add("v1").SetHeaders(
		map[string]interface{}{
			"Access-Control-Allow-Headers":		"Authorization,Content-Type",
			"Access-Control-Allow-Origin":		"*",
		},
	)

```

</div>
<div id="validation">

# Validation

<a href="#contents">Top</a>

One of the awesome things about JSONrouter is that every path-parameter & top-level JSON-object-parameter can have validation. This can be provided by one of many out-of-the-box validation functions, but custom validation functions can also be used with this system, allowing you to create the correct validation that your app needs. Validation functions can be thought of as middleware functions that can modify the request object by adding new params using req.SetParam(key, value), which means the validated values will always be referenceable through the request object.

A struct of type *validation.Config is passed to the .Param(...) method of any *web.Node to create a path parameter. This config struct has fields for path & JSON params to allow the config to be used to validate either.

```

func NewValidationConfig(validationType interface{}, pathFunction PathValidationFunction, bodyFunction BodyValidationFunction) *validation.Config {

	return &validation.Config{
		model: validationType,
		pathFunction: pathFunction,
		bodyFunction: bodyFunction,
	}
}

func Int() *validation.Config {

	min := 0.0
	max := 0.0

	return NewValidationConfig(
		0,
		func (web.http.Request, param string) (bool, interface{}) {

			if len(param) == 0 { return false, nil }

			val, err := strconv.Atoi(param)

			return err == nil, val
		},
		func (web.http.Request, param interface{}) (bool, interface{}) {

			i, ok := param.(float64)

			return ok, int(i)
		},
		min,
		max,
	)
}

```

</div>
<div id="JSONParam">

## JSON parameter validation

<a href="#contents">Top</a>

Using the .Payload method on any .POST(...) or .PUT(...) will allow the specification of the JSON request body payload. JSON objects can be described with a web.Payload (custom type equating to map[string]*validation.Config), which pairs keys with instances of the *validation.Config type. Each key will be validated with the func value of the bodyFunction field in the config struct.


```

	.POST(
		app.countryAddtown,
	).Required(
		&common.Payload{
			"country": validation.CountryISO2(),
			"tier":	validation.Int(),
		},
	)

```

When JSON params are added to the http.Request params map, they are prefixed with an underscore to avoid name collisions with route params. Below is an example of accessing the above payload parameters via an instance of http.Request.

```

	country := req.BodyParam("country").(*common.Country)
	town := req.BodyParam("town").(*custom.Entity)
	tier := req.BodyParam("tier").(int)

```

</div>
<div id="pathParam">

## Path parameter validation

<a href="#contents">Top</a>

### validation.String(...)

This function validates a req path param to a string. Passing a single integer to the function sets the explicit length of expected param, whereas passing 2 integers allows a string length to be between a min-max range. This function also sanitizes the input using the BlueMonday sanitization package.

```

.Param(validation.String(), "id") // any string

.Param(validation.String(30), "id") // string of length 30 chars

.Param(validation.String(3, 30), "id") // string with min length 3 chars, and max length 30 chars

```

### web.SplitString(...)

This function creates an array of string from a delimited string. A string argument will be used as a delimiter for strings.Split(...), creating an []string which is set to the request params. Each member of the array is sanitized using the BlueMonday sanitization package.

```

.Param(validation.SplitString(","), "modes")

```

### web.Int()

This function allows any 32-bit integer.

No arguments can be supplied to this function.

### web.Int64()

This function allows any 64-bit integer.

No arguments can be supplied to this function.

### web.Float64()

This function allows any 64-bit floating-point number.

No arguments can be supplied to this function.

### web.Bool()

This function allows either 'true' or 'false'.

No arguments can be supplied to this function.

### web.MapStringInterface()

This function allows a map[string]interface{} assertion from a JSON object.

No arguments can be supplied to this function.

```

	// JSON POST payload: {"myObject":{"name":"Hello world!"}}

	.POST(
		pp.DoSomething,
	).Required(
		&common.Payload{
			"myObject":	validation.MapStringInterface(),
		},
	)

```

### web.InterfaceArray()

This function allows an []interface{} assertion from a JSON array.

No arguments can be supplied to this function.

```

	// JSON POST payload: {"myArray":["Hello world!"]}

	.POST(
		app.DoSomething,
	).Required(
		common.Payload{
			"myArray": validation.InterfaceArray(),
		},
	)

```

### web.CountryISO2()

This function allows any ISO2 country code, which is resolved as a *web.Country struct (see common/countries.go).

No arguments can be supplied to this function.

### web.LanguageISO2()

This function allows any ISO2 language code, which is resolved as a *web.Language struct (see common/languages.go).

No arguments can be supplied to this function.

</div>
<div id="servingFiles">

# Serving files and folders

<a href="#contents">Top</a>

Serving files can be done by adding a folder path, or by specifying an explicit filepath. The content-type of the file is auto-detected.

```

	templates := root.Add("/templates")

		templates.Add("index.html").File("dst/pages/index.html")

```

or

```

	templates := root.Add("/templates").Folder("dst/pages")

```

In both cases the file would be reached with the path '/templates/index.html'.

The default behaviour is that the files & their content-types are cached the first time the file is requested. To disable this behaviour, the NoCache() method on the Config struct need to be called.

```

	root.Config.NoCache()

```

</div>
<div id="abstraction">

# Abstraction modules

<a href="#contents">Top</a>

Another cool feature of GF is that you can chain custom middleware functions together to allow the abstraction of common operations that your web application uses. To enable this feature, a module registry must be set to the GF config which maps the string key-names for modules to their web.ModuleFunction type instances.

An example of this is a piece of middleware that validates the user's session, and creates a context for the request.

```

	moduleRegistry := common.ModuleRegistry{
		"check_UserSession":		func(req http.Request, arg interface{}) *http.Status {

			hValue := req.GetHeader("Authorization")

			if hValue == "password" {
				req.SetParam("authenticated", true)
			} else {
				req.SetParam("authenticated", false)
			}

			return nil
		},
	}

	root.Config.SetModuleRegistry(moduleRegistry)

	root.Mod("check_UserSession", nil).GET(
		app/HandlerFunction,
	)

```

</div>
<div id="autoGen">

# Auto-generated documentation

<a href="#contents">Top</a>

A great feature of JSONrouter will create documentation based on the internal state of the router. This allows front-end developers access to a definitive resource describing the structure of your API. When initialising your app, you can choose OpenAPI spec either v2 or v3.

## JSON API schema

Currently JSONrouter will serve a documentation JSON file by default at the /openapi.spec.json path. This will describe each handler using any available info or validation descriptions, or the Response(...) method's descriptions.


</div>
