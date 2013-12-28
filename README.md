This package contains tools for helping you write [transforms](https://github.com/substack/node-browserify#btransformtr) for [browserify](https://github.com/substack/node-browserify).

Many different transforms perform certain basic functionality, such as turning the contents of a stream into a string, or loading configuration from package.json.  This package contains helper methods to perform these common tasks, so you don't have to write them over and over again:

* `makeStringTransform()` creates a transform which consumes and returns a string, instead of using a stream.
* `makeFalafelTransform()` parses a JS file using [falafel](https://github.com/substack/node-falafel) and allows you to modify the code.
* `makeRequireTransform()` passes you the contents of each `require()` call in each script, and allows you to rewrite the require statement.
* All of the above will automatically search for transform configuration in package.json and pass it to you if available, but if you have a more complicated use case than the `make*Transform()` functions will support, then `loadTransformConfig()` will load configuration for you.
* `runTransform()` can be used to unit test your shiny new transform.

Installation
============

Install with `npm install --save-dev browserify-transform-tools`.

Creating a String Transform
===========================
Browserify transforms work on streams.  This is all well and good, until you want to call a library like "falafel" which doesn't work with streams.  (If you're using falafel specifically, see below for `makeFalafelTransform`.)

Suppose you are writing a transform called "unbluify" which replaces all occurances of "blue" with a color loaded from a configuration:

```JavaScript
var options = {excludeExtensions: [".json"]};
module.exports = transformTools.makeStringTransform("unbluify", options,
    function (content, transformOptions, done) {
        var file = transformOptions.file;
        if(!transformOptions.config) {
            return done(new Error("Could not find unbluify configuration."));
        }

        done null, content.replace(/blue/g, transformOptions.config.newColor);
    });
```

Notice that the color we replace "blue" with gets loaded from configuration.  The configuration
can be set in a variety of ways.  A simple example is to set it directly in package.json:

```JavaScript
{
    "name": "myProject",
    "version": "1.0.0",
    ...
    "unbluify": {"newColor": "red"}
}
```

See the section on "Loading Configuration" below for details on where configuration can be loaded from.

Parameters for `makeStringTransform()`:

* `transformFn(contents, transformOptions, done)` - Function which is called to
  do the transform.  `contents` are the contents of the file.  `done(err, transformed)` is
  a callback which must be called, passing the a string with the transformed contents of the
  file.  transformOptions consists of:

  * `transformOptions.file` is the name of the file (as would be passed to a normal browserify transform.)
  * `transformOptions.configData` is the configuration data for the transform (see
  `loadTransformConfig` below for details on where this comes from.)
  * `transformOptions.config` is a copy of `transformOptions.configData.config` for convenience.

* `options.excludeExtensions` - A list of extensions which will not be processed.  e.g.
  "['.coffee', '.jade']"

* `options.includeExtensions` - A list of extensions to process.  If this options is not
  specified, then all extensions will be processed.  If this option is specified, then
  any file with an extension not in this list will skipped.

Creating a Falafel Transform
============================
Many transforms are based on [falafel](https://github.com/substack/node-falafel). browserify-transform-tools provides an easy way to define such transforms.  Here is an example which wraps all array expressions in a call to `fn()`:

```JavaScript
var options = {};
// Wraps all array expressions in a call to fn().  e.g. '[1,2,3]' becomes 'fn([1,2,3])'.
module.exports = transformTools.makeFalafelTransform("array-fnify", options,
    function (node, transformOptions, done) {
        if (node.type === 'ArrayExpression') {
            node.update('fn(' + node.source() + ')');
        }
        done();
    });
```

`makeFalafelTransform()` will be called once for every node in your JS file.  You can update the node.  Be sure to pass errors back via `done(err)`, and call `done()` when complete.

Options passed to `makeFalafelTransform()` are the same as for `makeStringTransform()`, as are the transformOptions passed to the transform function.  You can additionally pass a `options.falafelOptions` to `makeFalafelTransform` - this object will be passed as an options object directly to falafel.

Creating a Require Transform
============================

Many transforms are focused on transforming `require()` calls.  browserify-transform-tools has a solution for this:

```JavaScript
transform = transformTools.makeRequireTransform("requireTransform",
    {evaluateArguments: true},
    function(args, opts, cb) {
        if (args[0] === "foo") {
            return cb(null, "require('bar')");
        } else {
            return cb();
        }
    });
```

This will take all calls to `require("foo")` and transform them to `require('bar')`.  Note that makeRequireTransform can parse many simple expressions, so the above would succesfully parse `require("f" + "oo")`, for example.  Any expression involving core JavaScript, `__filename`, `__dirname`, `path`, and `join` (where join is an alias for `path.join`) can be parsed.  Setting the `evaluateArguments` option to false will disable this behavior, in which case the source code for everything inside the ()s will be returned.

Note that `makeRequireTransform` expects your function to return the complete `require(...)` call.  This makes it possible to write require transforms which will, for example, inline resources.

Again, all other options you can pass to `makeStringTransform` are valid here, too.

Loading Configuration
=====================

All `make*Transform()` functions will automatically load configuration for your transform and make it available via `transformOptions.configData` and `transformOptions.config`.

All `make*Transform()` functions return a transform which has a function called `configure(config, options)` which can be called to pass configuration directly to the transform.  `config` will be passed to the transform as `transformOptions.configData.config` and `transformOptions.config`.  If `options.configFile` is set, it will be used to set `transformOptions.configFile` and `transformOptions.configDir` - these are passed to transforms so they can resolve relative path names.  You can also specify `options.configDir` directly.  `configure()` returns a new transform instance and does not modify the existing transform.  If want to modify the configuration on an existing instance, you can call `setConfig()` with the same options.

If neither `configure()` nor `setConfig()` is called, then a transform will look for configuration in package.json.  For example, for a "unbluify" transform:

```JavaScript
{
    "name": "myProject",
    "version": "1.0.0",
    ...
    "unbluify": {"newColor": "red"}
}
```

Or alternatively you can set the "unbluify" key to be a js or JSON file:

```JavaScript
{
    "unbluify": "unbluifyConfig.js"
}
```

And then configuration will be loaded from that file:

```JavaScript
module.exports = {
    newColor: "red"
};
```

Note this means you can use enviroment variables to make changes to your configuration.

If you are writing your own transform which doesn't use a `make*Transform()` function, you can still use browserify-transform-tools to load configuration from package.json:

```JavaScript
var transformTools = require('browserify-transform-tools');

var configData = transformTools.loadTransformConfig('myTransform', file, function(err, configData) {
    var config = configData.config;
    var configDir = configData.configDir;
    ...
});

```

`loadTransformConfig()` will search the parent directory of `file` and its ancestors to find a `package.json` file.  Once it finds one, it will look for a key called 'myTransform' (taken from the transformName passed into `loadTransformConfig()`.)  If this key maps to a JSON object, then `loadTransformConfigSync()` will return the object.  If this key maps to a string, then `loadTransformConfigSync()` will try to load the JSON or JS file the string represents and will return that instead.  For example, if package.json contains `{"myTransform": "./myTransform.json"}`, then the contents of "myTransform.json" will be returned.  `configData.config` is the loaded data.  `configData.configDir` is the directory which contained the file that data was loaded from (handy for resolving relative path names.)  For other fields returned by `loadTransformConfigSync()`, see comments in [the source](https://github.com/benbria/browserify-transform-tools/blob/master/src/transformTools.coffee).

There is a synchronous version of this function, as well, called `loadTransformConfigSync(transformName, file)`.

Note that since configuration can be supplied in a .js file, the .js file can alter the configuration based on environment variables.

Running a Transform
===================
If you want to unit test your transform, then `runTransform()` is for you:

```JavaScript
var myTransform = transformTools.makeFalafelTransform(...);
var dummyJsFile = path.resolve(__dirname, "../testFixtures/testWithConfig/dummy.js");
var content = "console.log('Hello World!');";
transformTools.runTransform(myTransform, dummyJsFile, {content: content},
    function(err, transformed) {
        // Verify transformed is what we expect...
    }
);
```

Thanks
======
Some of this was heavily inspired by:

* [ForbesLindesay](https://github.com/ForbesLindesay)'s [rfileify](https://github.com/ForbesLindesay/rfileify)
* [thlorenz](https://github.com/thlorenz)'s [browserify-shim](https://github.com/thlorenz/browserify-shim)

