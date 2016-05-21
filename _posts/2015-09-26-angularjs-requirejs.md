---
title: AngularJS + RequireJS done right
---

Since AngularJS 2.x is not here yet, many applications are still being written using Angular 1.x. When the application that is more complex than the trivial examples you may found out there, it may end up having many modules, services, controllers, view templates and so on. In those real world cases, the main entry point of the application `index.html` may become a mess like this:

```HTML
<!DOCTYPE html>
<html ng-app="myApp">
	<head>
        <!-- Angular and modules -->
		<script src="path-to/angular.js"></script>
		<script src="path-to/angular-route.js"></script>
        <!-- more angular-related modules... -->
        <!-- more dependencies... -->
        <!-- more more and more and more... -->

        <!-- Lots of app files -->
		<script src="js/app.js"></script>
        <script src="js/service1.js"></script>
		<script src="js/service2.js"></script>
        <script src="js/controller1.js"></script>
        <script src="js/controller2.js"></script>
		<script src="js/controller3.js"></script>
        <!-- more script tags... -->
        <!-- and more script tags... -->
        <!-- and more and more and more... -->
	</head>
	<body>
		<ng-view></ng-view>
	</body>
</html>
```

Every time a new dependency or file has to be added to the application, the `index.html` must be updated with more and more `<script>` tags the old way, going back to 2009/2010. But in the last 5 years, many solutions to this problem have been created, like AMD and its de-facto implementation: RequireJS. Using RequireJS simplifies the development process taking care of the dependencies management and the `<script>` injection automatically so using it for complex applications is am must. Using other building mechanisms like Browserify or Webpack is also a step in the right direction, in opposition to stick to the old way of manually loading the scripts in the main HTML file.

The difficult of mixing AngularJS with AMD is in its very nature: AMD loads the scripts asynchronously but AngularJS requires the files that build the application to be loaded with a specific order respecting their own dependencies graph. For instance, and AngularJS module has to be created before any service or controller is defined on that module. RequireJS has settings to solve most of these issues but the application and its files must follow a proper pattern to let everything work as expected.

## Simplifying the main HTML file

Ideally the `index.html` will look like the following:

### `index.html`

```html
<!DOCTYPE html>
<html>
	<head>
		<script src="node_modules/requirejs/require.js" data-main="js/main.js"></script>
	</head>
	<body>
		<ng-view></ng-view>
	</body>
</html>

```

This is the minimum HTML file you may have. It loads `require.js` and then loads and executes the JavaScript entry point of the application `main.js`. No other script is loaded in the HTML, not even `angular.js`. The tags are also at the minimum: only an empty `ng-view` tag where the AngularJS app will be rendered.

## Loading AngularJS and its modules properly

AngularJS modules that extend the core functionality like `angular-route` and are distributed in their own files, must be loaded once `angular.js` is loaded. That can be done by setting the dependencies in the RequireJS configuration:

### `main.js`

```javascript
require.config({
    paths: {
        "angular": "path-to/angular",
        "angular-route": "path-to/angular-route",
        "domReady": "path-to/domReady"
    },
    shim: {
        "angular": {
            exports: "angular"
        },
        "angular-route": {
            deps: ["angular"]
        }
    }
});
```

The configuration declares AngularJS exports the global `angular` object and makes it available when `"angular"` is required. That prevents the modules to use the global object and use the parameter received in the `define()` declaration instead. Additionally, marks `angular-roture`, or any other module the application may use, as having `angular` as dependency. Therefore, any time `angular-route` is required, RequireJS will ensure `angular` is already loaded.

## Bootstrapping AngularJS

Before AngularJS is bootstrapped, the AngularJS components (configuration, modules, controllers, services) must be instantiated. Then, the DOM has to be completely loaded and finally the bootstrap process must be [initiated manually](https://docs.angularjs.org/guide/bootstrap).

To be sure the DOM is fully loaded, the loader plugin `domReady` could be used.

To be sure all components are loaded, those will be compiled in a `app.js` file.

### `main.js`

```javascript
require.config({
    // paths, shims and other options
})

require(["domReady!", "angular", "app"], function (doc, angular) {
    angular.bootstrap(document, ['myApp']);
});
```

The previous code does exactly what is needed: once the DOM is ready, RequireJS will load angular, all its components (compiled in `app.js`) and finally will execute the callback and bootstrap AngularJS.

## Organizing the AngularJS components

The script `app.js` has a single responsibility: to load all AngularJS components. Each component is either a service, a controller, a custom directive, that depends on the existence of an AngularJS module. That dependency can be solved by RequireJS automatically, so `app.js` will only contain the list of services to be instantiated. These components, in turn, will require the corresponding module which will require `angular` to complete the dependency chain.

First, the module has to be created. The module will require all its own dependencies like `ngRoute` in this case:

### `myAppModule.js`

```javascript
define([
    "angular",
    "angular-route"
], function (angular) {
    return angular.module("myApp", ["ngRoute"]);
});
```

Each component, as the following service, when loaded will require the module to be loaded and initialized first, so `app` will exist and the service will be attached properly:

### `myAppService1.js`

```javascript
define([
    "./myAppModule"
], function (app) {
    app.service("myAppService1", function (/* service dependencies */) {
        return {
            // service interface
        };
    });
});
```

Finally, all components combined will form the whole AngularJS application:

### `app.js`

```javascript
define([
    "./myAppService1",
    "./myAppService2",
    "./myAppController1",
    "./myAppController2",
    "./myAppController3"
], function () {
    // do nothing, just force to load the angular components
});
```

Notice that `app.js` does not need to load neither `angular` not the AngularJS modules. Those scripts will be automatically loaded as dependencies of the AngularJS components by RequireJS.

# Conclusion

Following the pattern described, it is possible to take advantage of dynamic and asynchronous script loading provided by RequireJS when developing an AngularJS app. That simplifies the development process as every component can live in its own file and all the dependencies are automatically managed by the loader.

Angular 2.x and ES2015 (aka ES6) may render all this obsolete but, until those technologies are mainstream, the proposed architecture will serve as a good starting point for developing web applications with the current and proved patterns.
