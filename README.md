undertaker
==========

[![Build Status](https://travis-ci.org/phated/undertaker.svg?branch=master)](https://travis-ci.org/phated/undertaker)

Task registry that allows composition through `series`/`parallel` methods.

## Usage

```js
var fs = require('fs');
var Undertaker = require('undertaker');

var taker = new Undertaker();

taker.task('task1', function(cb){
  // do things

  cb(); // when everything is done
});

taker.task('task2', function(){
  return fs.createReadStream('./myFile.js')
    .pipe(fs.createWriteStream('./myFile.copy.js'));
});

taker.task('task3', function(){
  return new Promise(function(resolve, reject){
    // do things

    resolve(); // when everything is done
  });
});

taker.task('combined', taker.series('task1', 'task2'));

taker.task('all', taker.parallel('combined', 'task3'));
```

## API

__Task functions can be completed in any of the ways supported by
[`async-done`](https://github.com/phated/async-done#completion-and-error-resolution)__

### `new Undertaker([RegistryConstructor])`

The constructor is used to create a new instance of `Undertaker`. Each instance of
`Undertaker` gets its own instance of a registry. By default, the registry is an
instance of [`undertaker-registry`](https://github.com/phated/undertaker-registry)
but it can be any other registry that follows the [Custom Registries API](#custom-registries).

To use a custom registry, pass the custom registry's constructor function when
instantiating a new `Undertaker` instance. This will use the custom constructor
to create the registry for this instance.

### `get(taskName)` => Function

Takes a string (`taskName`) representing the name of a register task and
returns the registered function.

### `set(taskName, fn)`

Takes a string (`taskName`) and a function (`fn`) to register as a task.
The `fn` gets registered in the registry by the `taskName`.

Will throw if:

* `taskName` is missing or not a string
* `fn` is missing or not a function

### `task([taskName,] fn)` => [Function]

An alias for `get` and `set`. If a string (`taskName`) is given as the only
argument, the `get` method will be called. If a function (`fn`) and optionally
a string (`taskName`) is given, the `set` method will be called.

This function allows you to pass a named function as the only argument and its
name will be turned into the `taskName` when the `set` method is called.

Will throw in the same ways mentioned in `get` and `set`.

### `series(taskName || fn...)` => Function

Takes a variable amount of strings (`taskName`) and/or functions (`fn`) and
returns a function of the composed tasks or functions. Any `taskNames` are
retrieved from the registry using the `get` method.

When the returned function is executed, the tasks or functions will be executed
in series, each waiting for the prior to finish. If an error occurs, execution
will stop.

### `parallel(taskName || fn...)` => Function

Takes a variable amount of strings (`taskName`) and/or functions (`fn`) and
returns a function of the composed tasks or functions. Any `taskNames` are
retrieved from the registry using the `get` method.

When the returned function is executed, the tasks or functions will be executed
in parallel, all being executed at the same time. If an error occurs, all execution
will complete.

### `registry([registryInstance])`

Optionally takes an instantiated registry object. If no arguments are passed, returns
the current registry object. If an instance of a registry (`customRegistry`) is passed
the tasks from the current registry will be transferred to it and the current registry
will be replaced with the new registry.

The ability to assign new registries will allow you to pre-define/share tasks or add
custom functionality to your registries. See [Custom Registries](#custom-registries)
for more information.

### `tree([options])` => Object

Optionally takes an object (`options`) and returns an object representing the
tree of registered tasks. The object returned is [`archy`](https://www.npmjs.org/package/archy)
compatible when assigned to the `nodes` key. Also, each node has a `type`
property that can be used to determine if the node is a `task` or `function`.

#### `options`

* `deep` - if the whole tree should be returned (Default: `false`)

## Custom Registries

Custom registries are constructor allowing you to pre-define/share tasks 
or add custom functionality to your registries.

A registries prototype should define:

- `get(taskName)`: returns the task with that name 
   or `undefined` if no task is registered with that name;
- `set(taskName, fn)`: add task to the registry;
- `tasks()`: returns an object listing all tasks in the registry.

The easiest way to create a custom registry is to inherits from 
`undertaker-registry`:

```javascript
var util = require('util');

var DefaultRegistry = require('undertaker-registry');

function MyRegistry(){
  DefaultRegistry.call(this);
}

util.inherits(MyRegistry, DefaultRegistry);

module.exports = MyRegistry;
```

### Sharing task

To share common tasks with all your projects, you can set a registry with those 
task pre-defined. For example you might want to share a `clean` task:

```javascript
var fs = require('fs');
var util = require('util');

var DefaultRegistry = require('undertaker-registry');
var del = require('del');

function CommonRegistry(){
  DefaultRegistry.call(this);

  var buildDir = './build';
  var exists = fs.existsSync(buildDir);

  if(exists){
    throw new Error('Cannot initialize common tasks. `build/` directory exists.');
  }

  this.set('clean', function(cb){
    del([buildDir], cb);
  });
}

util.inherits(CommonRegistry, DefaultRegistry);

module.exports = CommonRegistry;
```

Then to use it in any of a project:
```javascript
var Undertaker = require('undertaker');
var CommonRegistry = require('myorg-common-tasks');

var taker = new Undertaker(CommonRegistry);

taker.task('build', taker.series('clean', function build(cb) {
  // do things
  cb();
}));
```


### Sharing Functionalities

If you define tasks in many packages but would like them all to share some data, 
you can use a custom registry to bind them to those data:

```javascript
var util = require('util');

var Undertaker = require('undertaker');
var CommonRegistry = require('myorg-common-tasks');
var DefaultRegistry = require('undertaker-registry');

var BuildRegistery = require('./build.js');
var ServeRegistery = require('./serve.js');


function ConfigRegistry(config){
  DefaultRegistry.call(this);
  this.config = config;
}

util.inherits(ConfigRegistry, DefaultRegistry);

ConfigRegistry.prototype.set = function set(name, fn) {
  var task = this._tasks[name] = fn.bind(this.config);
  return task;
};

var taker = new Undertaker();

taker.registry(new BuildRegistery());
taker.registry(new ServeRegistery());

// `taker.registry` will reset each task in the registry with 
// `ConfigRegistry.prototype.set` which will bind them to the config object.
taker.registry(new ConfigRegistry({
  src: './src',
  build: './build',
  bindTo: '0.0.0.0:8888' 
}));

taker.task('default', taker.series('clean', 'build', 'serve', function default(cb) {
  console.log('Server bind to ' + this.bindTo);
  console.log('Serving' + this.build);
  cb();
}))
```

## License

MIT
