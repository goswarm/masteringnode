
# CommonJS Module System

[CommonJS](http://commonjs.org) is a community driven effort to standardize packaging of JavaScript libraries, known as _modules_. Ideally modules written in JavaScript which support the CommonJS module pattern can be re-used within many environments such as node, narwhal, and even browsers among others.

## Creating Modules

Lets create a utility module named _utilities_, which will contain a `merge()` function to copy the properties of one object to another. Typically in a browser, or environment without CommonJS module support, this may look similar to below, where `utils` is a global variable. Although namespacing can lower the chance of collisions, it can still become an issue, and when further namespacing is applied it can look flat-out silly.

    var utilities = {};
	utilities.merge = function(obj, other) {};

CommonJS modules remove this conflict by "wrapping" the contents of a JavaScript file with a closure similar to what is shown below, however more pseudo globals are available to the module, not just `exports`, `require`, and `module`. The `exports` object is then returned when a user invokes `require('utils')`.

    var module = { exports: {}};
	(function(module, exports){
	    exports.merge = function(){};
	})(module, module.exports);

First create the file _./utilities.js_, and define the `merge()` function below.

	exports.merge = function(obj, other) {
	    var keys = Object.keys(other);
	    for (var i = 0, len = keys.length; i < len; ++i) {
	        var key = keys[i];
	        obj[key] = other[key];
	    }
	    return obj;
	};

Next we will look at utilizing out new module in other libraries.

## Requiring Modules

There are four main ways to require a module in node, first is the _synchronous_ method, which simply returns the module's exports, second is the _asynchronous_ method which accepts a callback, third is the _asynchronous http_ method which can load remote modules, and lastly is requiring of shared libraries or "node addons" which we will cover later.

To get started create a second file named _./app.js_ with the code shown below. First we load in the core `sys` module, which provides common methods such as `sys.puts()`, `sys.print()`, and the object inspection method `sys.p()`. 

	var sys = require('sys'),
	    utils = require('./utilities');

	var a = { one: 1 };
	var b = { two: 2 };
	utils.merge(a, b);
	sys.p(a);

Since the `sys` module is bundled with node, it's `exports` are returned, however for other modules node will iterate the `require.paths` array in search of a module matching the given path. By default `require.paths` includes _~/.node_libraries_, so if we have _~/.node_libraries_/utilities.js_ we may simply `require('utilities')`, instead of our relative example `require('./utilities')` shown above.

Node also supports the idea of _index_ JavaScript files, to illustrate this example lets create a _math_ module that will provide the `math.add()`, and `math.sub()` methods. For organizational purposes we will keep each method in their respective _./math/add.js_ and _./math/sub.js_ files. So where does _index.js_ come into play? we can populate _./math/index.js_ with the code shown below, which is used when `require('./math')` is invoked, which is conceptually identical to invoking `require('./math/index')`.

	exports = {
	    add: require('./add'),
	    sub: require('./sub')
	};
	
The contents of _./math/add.js_ show us a new technique, here we use `module.exports` instead of `exports`. Previously mentioned was the fact that `exports` is not the only object exposed to the module file when evaluated, we also have access to `__dirname`, `__filename`, and `module` which represents the current module. Here we simply define the module export object to a new object, which happens to be a function. 

	module.exports = function add(a, b){
	    return a + b;
	};

This technique is usually only helpful when your module has one aspect that it wishes to expose, be it a single function, constructor, string, etc. Below is an example of how we could provide the `Animal` constructor:

    exports.Animal = function Animal(){};

which can then be utilized as shown:

    var Animal = require('./animal').Animal;

if we change our module slightly, we can remove `.Animal`:

    module.exports = function Animal(){};

which can now be used without the property:

    var Animal = require('./animal');

## Asynchronous Require

Node provides us with an asynchronous version of `require()`, aptly named `require.async()`. Below is the sample example previously shown for our _utilities_ module, however non blocking. `require.async()` accepts a callback of which the first parameter `err` is `null` or an instanceof `Error`, and then the module exports. Passing the error (if there is one) as the first argument is an extremely common idiom in node for async routines.
    
	require.async('sys', function(err, sys){
	    require.async('./utilities', function(err, utils){
	        sys.p(utils.merge({ foo: 'bar' }, { bar: 'baz' }));
	    });
	});

## Requiring Over HTTP

Asynchronous requires in node also have the added bonus of allowing module loading via **HTTP** and **HTTPS**.
To require a module via http all we have to do is pass a valid url as shown in the _sass_ to _css_ compilation example below: 

    
	var sys = require('sys'),
	    sassUrl = 'http://github.com/visionmedia/sass.js/raw/master/lib/sass.js',
	    sassStr = ''
	        + 'body\n'
	        + '  a\n'
	        + '    :color #eee';

	require.async(sassUrl, function(err, sass){
	    var str = sass.render(sassStr);
	    sys.puts(str);
	});

Outputs:

    body a {
	  color: #eee;}