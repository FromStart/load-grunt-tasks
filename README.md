# FromScratch - load-grunt-tasks

In this first article, I have decided to present a very simple open source project : **load-grunt-tasks**. This grunt plugin makes it possible to load dynamically all grunt plugins defined in the **package.json** file of your project.

You can find the original repository of this project on Github : [https://github.com/sindresorhus/load-grunt-tasks](https://github.com/sindresorhus/load-grunt-tasks).

When you want to use this plugin, according to the official documentation, you should add the next JavaScript code in your Grunt file :

```javascript
require('load-grunt-tasks')(grunt);
```

In order to follow this syntax, you should create, in a new javascript file, a new **NodeJS** module exporting a function. This function will take two parameters : the **grunt** object, and eventually a configuration object.

```javascript
module.exports = function (grunt, opts) {

}
```

Thanks to this configuration object, we will be able to define few things :
* **pattern** : an array of patterns defining which NPM modules should the plugin take into account.
* **scope** : from which dependency block, the grunt dependencies will be get from.
* **config** : the path to the package.json file, from where NPM dependencies will be fetched.
* **requireResolution** : boolean value used to defined with method from the Grunt API should be used for the load of the grunt tasks : **loadTasks** or **loadNpmTasks**.

We will first have a look to the way we will manage the configuration of the three first parameters. The **pattern** parameter will have the patterns grunt- and @\*/grunt-\* as a default value.

The default value for the **scope** parameter will be the four default dependencies blocks defined in the package.json file : *dependencies*, *devDependencies*, *peerDependencies* and *optionalDependencies*.

We will set the default value for the **config** file with the path to the first parent **package.json**. To get this path, we will use the external NPM module *pkg-up* and its *sync method*. This dependency has to be defined in the package.json of our module.

We will use also another NPM module, **arrify** to initialize array. This module is very simple, it will only convert non-array variable into array.

To define default value, we will use the pattern `var variable = option || defaultValue`.

```javascript
'use strict';
var pkgUp = require('pkg-up');
var arrify = require('arrify');

module.exports = function (grunt, opts) {
	opts = opts || {};

  var pattern = arrify(opts.pattern || ['grunt-*', '@*/grunt-*']);
	var config = opts.config || pkgUp.sync();
	var scope = arrify(opts.scope || ['dependencies', 'devDependencies', 'peerDependencies', 'optionalDependencies']);
}
```

Then we will resolve the full path to the config file, defined by the **config** parameter, and import it, in order to read all dependencies defined, for the previously defined **scope** parameter. To define the full path of this file, we will use the out-of-the-box **path** module and its **resolve** method.

```javascript
'use strict';
var path = require('path');
var pkgUp = require('pkg-up');
var arrify = require('arrify');

module.exports = function (grunt, opts) {
	opts = opts || {};

  var pattern = arrify(opts.pattern || ['grunt-*', '@*/grunt-*']);
	var config = opts.config || pkgUp.sync();
	var scope = arrify(opts.scope || ['dependencies', 'devDependencies', 'peerDependencies', 'optionalDependencies']);

  if (typeof config === 'string') {
		config = require(path.resolve(config));
	}
}
```

In order to avoid the load useless modules, we will add to the pattern array a specific syntax to exclude the **grunt** and **grunt-cli** modules. In order to exclude patterns, we will use the syntax **!name-of-what-you-want-to-exlude**. This pattern is usable with the **multimatch** module used in the next part.

```javascript
'use strict';
var path = require('path');
var pkgUp = require('pkg-up');
var arrify = require('arrify');

module.exports = function (grunt, opts) {
	opts = opts || {};

  var pattern = arrify(opts.pattern || ['grunt-*', '@*/grunt-*']);
	var config = opts.config || pkgUp.sync();
	var scope = arrify(opts.scope || ['dependencies', 'devDependencies', 'peerDependencies', 'optionalDependencies']);

  if (typeof config === 'string') {
		config = require(path.resolve(config));
	}

  pattern.push('!grunt', '!grunt-cli');
}
```

Before the last step, we will create an array, `names`, containing all NPM modules, matching all patterns defined previously. For each scope, we will concatenate to this array all these modules, thanks to the **concat** method. This method can only take an array as a first argument. In order to avoid issue, we will first check if the value we use is an array, if not (if it is in fact an object), we will convert it to an array with the **Object.keys** method.

```javascript
'use strict';
var path = require('path');
var pkgUp = require('pkg-up');
var arrify = require('arrify');

module.exports = function (grunt, opts) {
	opts = opts || {};

  var pattern = arrify(opts.pattern || ['grunt-*', '@*/grunt-*']);
	var config = opts.config || pkgUp.sync();
	var scope = arrify(opts.scope || ['dependencies', 'devDependencies', 'peerDependencies', 'optionalDependencies']);

  if (typeof config === 'string') {
		config = require(path.resolve(config));
	}

  pattern.push('!grunt', '!grunt-cli');

  var names = scope.reduce(function (result, prop) {
		var deps = config[prop] || [];
		return result.concat(Array.isArray(deps) ? deps : Object.keys(deps));
	}, []);
}
```

We have right now the list of all modules defined in our package.json, we have now to be sure they respect the pattern we have define with the **pattern** variable. For this task, we will use the external **multimatch** module. Its default method takes two parameters : the list of modules, and the pattern. The returned value will be an array, and we will loop through each value of this array, in order to load the corresponding module in the *Grunt** execution context.

```javascript
'use strict';
var path = require('path');
var pkgUp = require('pkg-up');
var multimatch = require('multimatch');
var arrify = require('arrify');

module.exports = function (grunt, opts) {
	opts = opts || {};

  var pattern = arrify(opts.pattern || ['grunt-*', '@*/grunt-*']);
	var config = opts.config || pkgUp.sync();
	var scope = arrify(opts.scope || ['dependencies', 'devDependencies', 'peerDependencies', 'optionalDependencies']);

  if (typeof config === 'string') {
		config = require(path.resolve(config));
	}

  pattern.push('!grunt', '!grunt-cli');

  var names = scope.reduce(function (result, prop) {
		var deps = config[prop] || [];
		return result.concat(Array.isArray(deps) ? deps : Object.keys(deps));
	}, []);

  multimatch(names, pattern).forEach(function (pkgName) {
  });
}
```

To finish, we have to load all modules available in the previously **names** array. According to the  **requireResolution** option, we will use two different methods defined in the Grunt API : **loadTask** or **loadNpmTasks**. The easiest one is **loadNpmTasks**. We just need to pass as a parameter the name of the module. For **loadTask**, we have to pass the path to a specific directory, commonly called **tasks** in the **GruntJS** environment. To resolve this path, we will use the last external module : **resolve-pkg**.

To conclude this very first article, you will find below the final result of this grunt plugin.

```javascript
'use strict';
var path = require('path');
var pkgUp = require('pkg-up');
var multimatch = require('multimatch');
var arrify = require('arrify');
var resolvePkg = require('resolve-pkg');

module.exports = function (grunt, opts) {
	opts = opts || {};

	var pattern = arrify(opts.pattern || ['grunt-*', '@*/grunt-*']);
	var config = opts.config || pkgUp.sync();
	var scope = arrify(opts.scope || ['dependencies', 'devDependencies', 'peerDependencies', 'optionalDependencies']);

	if (typeof config === 'string') {
		config = require(path.resolve(config));
	}

	pattern.push('!grunt', '!grunt-cli');

	var names = scope.reduce(function (result, prop) {
		var deps = config[prop] || [];
		return result.concat(Array.isArray(deps) ? deps : Object.keys(deps));
	}, []);

	multimatch(names, pattern).forEach(function (pkgName) {
		if (opts.requireResolution === true) {
			try {
				grunt.loadTasks(resolvePkg(path.join(pkgName, 'tasks')));
			} catch (err) {
				grunt.log.error('npm package "' + pkgName + '" not found. Is it installed?');
			}
		} else {
			grunt.loadNpmTasks(pkgName);
		}
	});
};

```
