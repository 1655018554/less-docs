Released [v2.5.0]({{ less.master.url }}CHANGELOG.md)

> Import JavaScript plugins to add Less.js functions and features

## Writing your first plugin

Using a `@plugin` at-rule is similar to using an `@import` for your `.less` files.
```less
@plugin "my-plugin";  // automatically appends .js if no extension
```
Less.js Plugins are (modifed) UMD (universal module definition)-format JavaScript files. The basic guts of a plugin looks like:
```js
(function (root, registerPlugin) { 
    if (typeof define === 'function' && define.amd) { 
        define([], registerPlugin);
    } else if (typeof module === 'object' && module.exports) { 
        module.exports = registerPlugin();
    } else { 
        if (!root.less) { root.less = {}; } 
        if (!root.less.plugins) { root.less.plugins = []; }
        root.less.plugins.push(registerPlugin());
    } 
}(this, function () {
    // Less.js Plugin object
    return {
        install: function(less, pluginManager, functions) {
            // functions.add('')
        }
    };
}));
```
Most of that is boilerplate UMD. The only part you need to modify is the returned plugin object.

_Note: Less.js Plugins should have this signature to be compatible with Less 3.x features moving forward, but Less 3.x is backwards-compatible with existing Less 2.x plugins._

What can you do with a plugin? A lot, but let's start with the basics. We'll focus first on what you might put inside the `install` function. Let's say you write this:

```js
// my-plugin.js
// ...boilerplate UMD
install: function(less, pluginManager, functions) {
    functions.add('pi', function() {
        return Math.PI;
    });
}
// etc
```
Congratulations! You've written a Less plugin! 

If you were to use this in your stylesheet:
```less
@plugin "my-plugin";
.show-me-pi {
  value: pi();
}
```
You would get:
```less
.show-me-pi {
  value: 3.141592653589793;
}
```
However, you would need to return a proper Less node if you wanted to, say, multiply that against other values or do other Less operations. Otherwise the output in your stylesheet is plain text (which may be fine for your purposes).

Meaning, this is more correct:
```js
functions.add('pi', function() {
    return less.Dimension(Math.PI);
});
```
_Note: A dimension is a number with or without a unit, like "10px", which would be `less.Dimension(10, "px")`. For a list of units, see the [Less API](TODO)._

Now you can use your function in operations.
```less
@plugin "my-plugin";
.show-me-pi {
  value: pi() * 2;
}
```

You may have noticed that there are available globals for your plugin file, namely a function registry (`functions` object), and the `less` object. These are there for convenience.


## Plugin Scope

Functions added by a `@plugin` at-rule adheres to Less scoping rules. This is great for Less library authors that want to add functionality without introducing naming conflicts.

For instance, say you have 2 plugins from two third-party libraries that both have a function named "foo".
```js
// lib1.js
// ...
    functions.add('foo', function() {
        return "foo";
    });
// ...

// lib2.js
// ...
    functions.add('foo', function() {
        return "bar";
    });
// ...
```
That's ok! You can choose which library's function creates which output.
```less
.el-1 {
    @plugin "lib1";
    value: foo();
}
.el-2 {
    @plugin "lib2";
    value: foo();
}
```
This will produce:
```less
.el-1 {
    value: foo;
}
.el-2 {
    value: bar;
}
```

For plugin authors sharing their plugins, that means you can also effectively make private functions by placing them in a particular scope. As in, this will cause an error:
```less
.el {
    @plugin "lib1";
}
@value: foo();
```

As of Less 3.0, functions can return any kind of Node type, and can be called at any level.

Meaning, this would throw an error in 2.x, as functions had to be part of the value of a property or variable assignment:
```less
.block {
    color: blue;
    my-function-rules();
}
```
In 3.x, that's no longer the case, and functions can return At-Rules, Rulesets, any other Less node, strings, and numbers (the latter two are converted to Anonymous nodes).

## Null Functions

There are times when you may want to call a function, but you don't want anything output (such as storing a value for later use). In that case, you just need to return `false` from the function.
```js
var collection = [];

functions.add('store', function(val) {
    collection.push(val);  // imma store this for later
    return false;
});
```
```less
@plugin "collections";
@var: 32;
store(@var);
```
Later you could do something like:
```js
functions.add('retrieve', function(val) {
    return less.Value(collection);
});
```
```less
.get-my-values {
    @plugin "collections";
    values: retrieve();   
}
```

## Registering a plugin globally 
(TODO: rewrite with UMD spec)

Plugins can optionally register themselves in the global scope (even if called within a block scope) by exporting an object with a specific signature. All properties of this object are optional, and will only be called if present. It can be returned with a `registerPlugin({...})` call, or you can use the CommonJS `module.exports = {...}` pattern.

_Note: while you can use `module.exports`, other Node.js functionality like `require()` is not provided by Less.js. It's recommended that plugin authors create single-file modules (or single-file builds) of plugins for cross-platform functionality._

The exported object can have any (or none) of these properties.
```js
{
    /* Called immediately after the plugin is 
     * first imported, only once. */
    install: function(less, pluginManager) { },

    /* Called for each instance of your @plugin. */
    use: function(context) { },

    /* Called for each instance of your @plugin, 
     * when rules are being evaluated.
     * It's just later in the evaluation lifecycle */
    eval: function(context) { },

    /* Passes an arbitrary string to your plugin 
     * e.g. @plugin (args) "file";
     * This string is not parsed for you, 
     * so it can contain anything */
    setOptions: function(argumentString) { },

    /* Set a minimum Less compatibility string
     * You can also use an array, as in [3, 0] */
    minVersion: ['3.0'],

    /* Used for lessc plugins only, to explain 
     * options in a Terminal */
    printUsage: function() { },

}
```
The PluginManager instance for the `install()` function provides methods for adding visitors, file managers, and post-processors.

Here are some example repos showing the different plugin types. <!-- TODO: updated examples -->
 - post-processor: https://github.com/less/less-plugin-clean-css
 - visitor: https://github.com/less/less-plugin-inline-urls
 - file-manager: https://github.com/less/less-plugin-npm-import

## Pre-Loaded Plugins

While a `@plugin` call works well for most scenarios, there are times when you might want to load a plugin before parsing starts.

See: [Pre-Loaded Plugins](/usage/#plugins) in the "Using Less.js" section for how to do that.
