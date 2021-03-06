angular.haxe
============

angular.haxe is a toolset for integration of angular.js in haxe applications. It tries to combine angular.js with the type safety of haxe as much as possible. The angular API is fully typed and haxe macros extend the api to support dependency injection based on types instead of unsafe strings.

It's currently incomplete, feel free to add pull requests or become a contributor.




# Example of angular.haxe dependency injection

```haxe
// a regular haxe model
class Config {
	public var appName = "News App";
	public function new () {}
}

class Main
{
	static function main ()
	{
		// controller definition, the parameter types determine
		// which dependencies needs to be injected.

		function appController (scope:Scope, config:Config) {
			trace("appController initialized");
			scope.set("data", { appName : config.appName });
		};

		// module definition

		var module =
			Angular.module("myModule", [])
			// no need give the factory a name,
			// the link is created by the types
			.factory(Config.new)
			.controller("AppController", appController);
	}
}
```
```html
<html>
<head>
	<title>Angular-Haxe</title>
</head>
<body ng-app="myModule" ng-controller="AppController">
	<h1>{{data.appName}}</h1>
</body>
</html>
```

See [Sample 2](sample/sample-02) for a slightly larger working example. The generating js file for this example can be found [here](sample/sample-02/bin/main.js).

# angular.haxe vs angular.js

here are same snippets which highlight differences of angular.haxe vs pure angular.js

## Factory / Service registration with dependencies

Factory registration with js.

```js

function MyService (timeout, location) {
	// do something with timeout and location
}

// register as factory
angular.module("myModule", [])
	.factory("myService", ["$timeout", "$location", function (timeout, location) {
		return new MyService(timeout, location);
	}])

```
In angular.haxe dependency injection in general is based on the type to achieve more type safety. This type gets translated into a string at compile time, e.g. my.pack.MyService becomes "my.pack.MyService". This translation is done through macros which run at compile time and convert the abstract syntax tree in such a way that the generated code is very similar to the js code.

```haxe
import angular.service.Timeout;
import angular.service.Location;
import angular.Angular;

class MyService {
	public function new (timeout:Timeout, location:Location) {
		// do something with timeout and location
	}
}

// register as factory
Angular.module("myModule", [])
	.factory(MyService.new);
```
This generated js code looks similar to the next snippet.
```js

function MyService { .... }

Angular.module("myModule", [])
	.factory("MyService", ["$timeout", "$location", function (a1, a2) {
		return new MyService(a1,a2)
	}]);
```

You may wonder why a type of a core angular services like Timeout are translated to $timeout instead of "angular.service.Timeout". The reason for this is that the core types are annotated with speacial metadata like @:injectionName("$timeout"). Types with this metadata are handled in special way inside of macros like factory, the value inside of this metadata is used instead of the full qualified class name.


## Controller

Controller in angular.js

```js
angular.module("myModule", [])
	.factory("myService", ["$timeout", "$location", function (timeout, location) {
		return new MyService(timeout, location);
	}])
	.controller("MyController", ["$scope", "myService", function (scope, myService) {
		// do something in your controller
	}])
```

The same logic in haxe

```haxe
Angular.module("myModule", [])
	.factory(MyService.new);
	.controller("MyController", function (scope:Scope, myService:MyService) {
		// do something
	})
```

## Directives

Directive in angular.js

```js
angular.module("myModule", [])
	.directive('myDialog', function() {
	    return {
	      restrict: 'E',
	      transclude: true,
	      scope: {},
	      templateUrl: 'my-dialog.html',
	      link: function (scope, element) {
	        scope.name = 'Jeff';
	      }
	    };
	  });

```

While you can define the directive in the same way (very dynamic), you can also use the class `angular.support.DirectiveBuilder` to create a directive. This class is a DSL to build a directive step by step. Note that the link function requires all five supported parameters instead of just the 2 like in the js version. But you can also write `function (scope:Scope, e:Element, _,_,_) {` if you don't require the last parameters.

```haxe
Angular.module("myModule", [])
	.directive('myDialog',
		DirectiveBuilder.mk()
			.restrictToElement()
			.transclude(true)
			.isolatedScope()
			.templateUrl('my-dialog.html')
			.link(function (scope:Scope, e:Element, a:Attributes, c:Controller, t:TranscludeFn) {
				scope.set("name", "Jeff");
			})
			.build
	);

```

# Advanced Topics

## Typed Scope

```haxe
import angular.*;
import angular.service.*;

typedef AppScope = { data : { firstName : String, lastName : String }};

class Main
{
	static function main ()
	{
		function appController (t:TypedScope<AppScope>) {
			t.data = {
				lastName : "tom",
				firstName : "timmy"
			};
		}
		var module =
			Angular.module("myModule", [])
			.controller("AppController", appController);


	}
}
```

## Using the same type for multiple services

In order to use the same type for multiple services typedefs (aliases) can be used. This is possible because this library uses macros to inspect the injected types at compilation time and not at runtime.

```haxe
import angular.*;
import angular.service.*;

typedef Url = String;
typedef Title = String;
typedef Description = String;

typedef AppScope = {
	data : { url : String, title : String, description : String }
};

class Main
{
	static function main ()
	{
		var module =
			Angular.module("myModule", [])
			.factory( function ():Url return "https://github.com/frabbit/angular.haxe")
			.factory( function ():Title return "angular.haxe")
			.factory( function ():Description return "Some words about angular.haxe...")
			.controller("AppController", function (s:TypedScope<AppScope>, url:Url, title : Title, desc:Description) {
				s.data = { url : url, title : title, description : desc };
			});
	}
}
```

As an alternative the special `Const` type parameter can be used to distinguish between different Strings by using constant string type parameters.

```haxe
import angular.*;
import angular.service.*;

typedef CustomString<Const> = String;

typedef AppScope = {
	data : { url : String, title : String, description : String }
};

class Main
{
	static function main ()
	{


		var module =
			Angular.module("myModule", [])
			.factory( function ():CustomString<"url"> return "https://github.com/frabbit/angular.haxe")
			.factory( function ():CustomString<"title"> return "angular.haxe")
			.factory( function ():CustomString<"desc"> return "Some words about angular.haxe...")


			.controller("AppController", function (s:TypedScope<AppScope>, url:CustomString<"url">,
				title : CustomString<"title">, desc:CustomString<"desc">) {

				s.data = { url : url, title : title, description : desc };
			});


	}
}
```

## Functions can be services and are identified by signature

```haxe
import angular.*;
import angular.service.*;

typedef Title = String;

typedef AppScope = {
	data : { title : String }
};

class Main
{
	static function main ()
	{
		var module =
			Angular.module("myModule", [])
			.factory( function ():Void->Title return function () return "angular.haxe")
			.factory( function ():Title->String return function (t:Title) return "!!! " + t + " !!!")
			.controller("AppController", function (s:TypedScope<AppScope>, getTitle:Void->Title, format:Title->String) {
				s.data = { title : format(getTitle()) };
			});
	}
}
```

## Bundles are auto-composed injectable structures based on other services

```haxe
package ;

import angular.*;
import angular.service.*;

typedef Bundle = {
	x : String,
	y : Int
};

typedef AppScope = {
	data : { title : String }
};

class Main
{
	static function main ()
	{
		var module =
			Angular.module("myModule", [])
			.factory( function () return 5)
			.factory( function () return "times")
			.bundle( ( _ : Bundle ) )
			.controller("AppController", function (s:TypedScope<AppScope>, bundle:Bundle) {
				s.data = { title : bundle.y + " " + bundle.x };
			});
	}
}
```

Technically `.bundle( ( _ : Bundle ) )` this is just a shortcut for the following code. It becomes really useful for larger bundles with a lot of fields.

```haxe
.factory( function (x:String, y:Int):Bundle {
	return { x : x, y : y}
})
```

