Express3 Handlebars
===================

A [Handlebars][] view engine for [Express][] which doesn't suck.

[![Dependency Status](https://david-dm.org/ericf/express3-handlebars.png)][status]


[Express]: https://github.com/visionmedia/express
[Handlebars]: https://github.com/wycats/handlebars.js
[status]: https://david-dm.org/ericf/express3-handlebars


Goals & Design
--------------

I created this project out of frustration with the existing Handlebars view
engines for Express. As of version 3.x, Express got out of the business of being
a generic view engine — this was a great decision — leaving developers to
implement the concepts of layouts, partials, and doing file I/O for their
template engines of choice.

### Goals and Features

After building a half-dozen Express apps, I developed requirements and opinions
about what a Handlebars view engine should provide and how it should be
implemented. The following is that list:

* Add back the concept of "layout", which was removed in Express 3.x.

* Add back the concept of "partials" via Handlebars' partials mechanism.

* Support a directory of partials; e.g., `{{> foo/bar}}` which exists on the
  file system at `views/partials/foo/bar.handlebars` by default.

* Smart file system I/O and template caching. When in development, templates are
  always loaded from disk. In production, raw files and compiled templates are
  cached, including partials.

* All async and non-blocking. File system I/O is slow and servers should not be
  blocked from handling requests while reading from disk. I/O queuing is used to
  avoid doing unnecessary work.

* Ability to expose precompiled templates and partials to the client, enabling
  template sharing and reuse.

* Ability to use a different Handlebars module/implementation other than the
  Handlebars npm module.

### Module Design

This module was designed to work great for both the simple and complex use
cases. I _intentionally_ made sure the full implementation is exposed and is
easily overrideable.

The module exports a function which can be invoked with no arguments or with a
`config` object and it will return a function (closed over sane defaults) which
can be registered with an Express app. It's an engine factory function.

This exported engine factory has two properties which expose the underlying
implementation:

* `ExpressHandlebars()`: The constructor function which holds the internal
  implementation on its `prototype`. This produces instance objects which store
  their configuration, `compiled` and `precompiled` templates, and expose an
  `engine()` function which can be registered with an Express app.

* `create()`: A convenience factory function for creating `ExpressHandlebars`
  instances.

An instance-based approach is used so that multiple `ExpressHandlebars`
instances can be created with their own configuration, templates, partials, and
helpers.


Installation
------------

Install using npm:

```shell
$ npm install express3-handlebars
```


Usage
-----

This module uses sane defaults that leverage the "Express-way" of structuring an
app's views. This makes it trivial to use this module in basic apps:

#### Basic Usage

**Directory Structure:**

```
.
├── app.js
└── views
    ├── home.handlebars
    └── layouts
        └── main.handlebars

2 directories, 3 files
```

**app.js:**

Creates a super simple Express app which shows the basic way to register a
Handlebars view engine using this module.

```javascript
var express = require('express'),
    exphbs  = require('express3-handlebars'),

    app = express();

app.engine('handlebars', exphbs({defaultLayout: 'main'}));
app.set('view engine', 'handlebars');

app.get('/', function (req, res, next) {
    res.render('home');
});

app.listen(3000);
```

**views/layouts/main.handlebars:**

The main layout is the HTML page wrapper which can be reused for the different
views of the app. `{{{body}}}` is used as a placeholder for where the main
content should be rendered.

```html
<!doctype html>
<html>
<head>
    <meta charset="utf-8" />
    <title>Example App</title>
</head>
<body>

    {{{body}}}

</body>
</html>
```

**views/home.handlebars:**

The content for the app's home view which will be rendered into the layout's
`{{{body}}}`.

```html
<h1>Example App: Home</h1>
```

### Using Instances

Another way to use this module is to create an instance(s) of
`ExpressHandlebars`, allowing access to the full API:

```javascript
var express = require('express'),
    exphbs  = require('express3-handlebars'),

    app = express(),
    hbs = exphbs.create({ /* config */ });

// Register `hbs.engine` with the Express app.
app.engine('handlebars', hbs.engine);
app.set('view engine', 'handlebars');

// ...still have a reference to `hbs`, on which methods like `loadPartials()`
// can be called.
```

### Template Caching

This module uses a smart template caching strategy. In development, templates
will always be loaded from disk, i.e., no caching. In production, raw files and
compiled Handlebars templates are aggressively cached.

The easiest way to control template/view caching is through Express'
[view cache setting][]:

```javascript
app.enable('view cache');
```

Express enables this setting by default when in production mode, i.e.,
`process.env.NODE_ENV === "production"`.

**Note:** All of the public API methods accept `options.cache`, which gives
control over caching when calling these methods directly.

### Layouts

A layout is simply a Handlebars template with a `{{{body}}}` placeholder.
Usually it will be an HTML page wrapper into which views will be rendered.

This module adds back the concept of "layout", which was removed in Express 3.x.
This view engine can be configured with a path to the layouts directory, by
default it's set to `"views/layouts/"`.

There are two ways to set a default layout: configuring the view engine's
`defaultLayout` property, or setting [Express locals][] `app.locals.layout`.

The layout into which a view should be rendered can be overridden per-request
by assigning a different value to the `layout` request local. The following
will render the "home" view with no layout:

```javascript
app.get('/', function (req, res, next) {
    res.render('home', {layout: false});
});
```

### Helpers

Helper functions, or "helpers" are functions that can be
[registered with Handlebars][] and can be called within a template. Helpers can
be used for transforming output, iterating over data, etc. To keep with the
spirit of *logic-less* templates, helpers are the place where logic should be
defined.

Handlebars ships with some [built-in helpers][], such as: `with`, `if`, `each`,
etc. Most application will need to extend this set of helpers to include
app-specific logic and transformations. Beyond defining global helpers on
`Handlebars`, this module supports `ExpressHandlebars` instance-level helpers
via the `helpers` configuration property, and render-level helpers via
`options.helpers` when calling the `render()` and `renderView()` methods.

The following example shows helpers being specified at each level:

**app.js:**

Creates a super simple Express app which shows the basic way to register
`ExpressHandlebars` instance-level helpers, and overide one at the render-level.

```javascript
var express = require('express'),
    exphbs  = require('express3-handlebars'),

    app = express(),
    hbs;

hbs = exphbs.create({
    // Specify helpers which are only registered on this instance.
    helpers: {
        foo: function () { return 'FOO!'; }
        bar: function () { return 'BAR!'; }
    }
});

app.engine('handlebars', hbs.engine);
app.set('view engine', 'handlebars');

app.get('/', function (req, res, next) {
    res.render('home', {
        showTitle: true,

        // Override `foo` helper only for this rendering.
        helpers: {
            foo: function () { return 'foo.'; }
        }
    });
});

app.listen(3000);
```

**views/home.handlebars:**

The app's home view which uses helper functions to help render the contents.

```html
<!doctype html>
<html>
<head>
    <meta charset="utf-8" />
    <title>Example App - Home</title>
</head>
<body>

    <!-- Uses built-in `if` helper. -->
  {{#if showTitle}}
    <h1>Home</h1>
  {{/if}}

    <!-- Calls `foo` helper, overridden at render-level. -->
    <p>{{foo}}</p>

    <!-- Calls `bar` helper, defined at instance-level. -->
    <p>{{bar}}</p>

</body>
</html>
```

#### More on Helpers

Refer to the [Handlebars website][] for more information on defining helpers:

* [Expression Helpers][]
* [Block Helpers][]


[view cache setting]: http://expressjs.com/api.html#app-settings
[Express locals]: http://expressjs.com/api.html#app.locals
[registered with Handlebars]: https://github.com/wycats/handlebars.js/#registering-helpers
[built-in helpers]: http://handlebarsjs.com/#builtins
[Handlebars website]: http://handlebarsjs.com/
[Expression Helpers]: http://handlebarsjs.com/expressions.html#helpers
[Block Helpers]: http://handlebarsjs.com/block_helpers.html


API
---

### Configuration and Defaults

There are two main ways to use this module: via its engine factory function, or
creating `ExpressHandlebars` instances; both use the same configuration
properties and defaults.

```javascript
var exphbs = require('express3-handlebars');

// Using the engine factory:
exphbs({ /* config */ });

// Create an instance:
exphbs.create({ /* config */ });
```

The following is the list of configuration properties and their default values
(if any):

#### `defaultLayout`
The string name or path of a template in the `layoutsDir` to use as the default
layout. This is overridden by a `layout` specified in the app or response
`locals`. **Note:** A falsy value will render without a layout; e.g.,
`res.render('home', {layout: false});`.

#### `extname=".handlebars"`
The string name of the file extension used by the templates.

#### `handlebars=require('handlebars')`
The Handlebars module/implementation. This allows for the `ExpressHandlebars`
instance to use a different Handlebars module/implementation than that provided
by the Handlebars npm module.

#### `helpers`
An object which holds the helper functions used when rendering templates with
this `ExpressHandlebars` instance. When rendering a template, a collection of
helpers will be generated by merging: `handlebars.helpers` (global), `helpers`
(instance), and `options.helpers` (render-level). This allows Handlebars'
`registerHelper()` function to operate as expected, will providing two extra
levels over helper overrides.

#### `layoutsDir="views/layouts/"`
The string path to the directory where the layout templates reside.

#### `partialsDir="views/partials/"`
The string path to the directory where the partials templates reside.

### Properties

The public API properties are provided via `ExpressHandlebars` instances. In
additional to the properties listed in the **Configuration and Defaults**
section, the following are additional public properties:

#### `compiled`
An object cache which holds compiled Handlebars template functions in the
format: `{"path/to/template": [Function]}`.

#### `engine`
A function reference to the `renderView()` method which is bound to `this`
`ExpressHandlebars` instance. This bound function should be used when
registering this view engine with an Express app.

#### `handlebarsVersion`
The version number of `handlebars` as a semver. This is unsed internally to
branch on certain operations which differ between Handlebars releases.

#### `precompiled`
An object cache which holds precompiled Handlebars template strings in the
format: `{"path/to/template": [String]}`.

### Methods

The following is the list of public API methods provided via `ExpressHandlebars`
instances:

#### `loadPartials(options|callback, [callback])`

Retrieves the partials in the `partialsDir` and passes an object mapping the
partials in the form `{name: partial}` to the `callback`.

By default each partial will be a compiled Handlebars template function. Use
`options.precompiled` to receive the partials as precompiled templates — this is
useful for sharing templates with client code.

**Parameters:**

* `[options]`: Optional object containing any of the following properties:

  * `[cache]`: Whether cached templates can be used if they have already been
    requested. This is recommended for production to avoid unnecessary file I/O.

  * `[precompiled=false]`: Whether precompiled templates should be provided,
    instead of compiled Handlebars template functions.

* `callback`: Function to call once the partials are retrieved.

The name of each partial corresponds to its location in `partialsDir`. For
example, consider the following directory structure:

```
views
└── partials
    ├── foo
    │   └── bar.handlebars
    └── title.handlebars

2 directories, 2 files
```

`loadPartials()` would produce the following result:

```javascript
var hbs = require('express3-handlebars').create();

hbs.loadPartials(function (err, partials) {
    console.log(partials);
    // => { 'foo.bar': [Function],
    // =>    title: [Function] }
});
```

**Note:** The partial name `"foo.bar"` would ideally be `"foo/bar"`, but this is
being prevented by a [Handlebars bug][]. Once this bug is fixed, a future
version will use a "/" separator. Templates requiring the partial still use:
`{{> foo/bar}}`.

#### `loadTemplate(filePath, options|callback, [callback])`

Retrieves the template at the specified `filePath` and passes a compiled
Handlebars template function to the `callback`.

Use `options.precompiled` to receive a precompiled Handlebars template.

**Parameters:**

* `filePath`: String path to the Handlebars template file.

* `[options]`: Optional object containing any of the following properties:

  * `[cache]`: Whether a cached template can be used if it have already been
    requested. This is recommended for production to avoid necessary file I/O.

  * `[precompiled=false]`: Whether a precompiled template should be provided,
    instead of a compiled Handlebars template function.

* `callback`: Function to call once the template is retrieved.

#### `loadTemplates(dirPath, options|callback, [callback])`

Retrieves the all the templates in the specified `dirPath` and passes an object
mapping the compiled templates in the form `{filename: template}` to the
`callback`.

Use `options.precompiled` to receive precompiled Handlebars templates — this is
useful for sharing templates with client code.

**Parameters:**

* `dirPath`: String path to the directory containing Handlebars template files.

* `[options]`: Optional object containing any of the following properties:

  * `[cache]`: Whether cached templates can be used if it have already been
    requested. This is recommended for production to avoid necessary file I/O.

  * `[precompiled=false]`: Whether precompiled templates should be provided,
    instead of a compiled Handlebars template function.

* `callback`: Function to call once the templates are retrieved.

#### `render(filePath, options|callback, [callback])`

Renders the template at the specified `filePath` using this instance's `helpers`
and partials, and passes the resulting string to the `callback`.

The `options` will be used both as the context in which the Handlebars template
is rendered, and to signal this view engine on how it should behave, e.g.,
`options.cache = false` will load _always_ load the templates from disk.

**Parameters:**

* `filePath`: String path to the Handlebars template file.

* `[options]`: Optional object which will serve as the context in which the
  Handlebars template is rendered. It may also contain any of the following
  properties which affect this view engine's behavior:

  * `[cache]`: Whether a cached template can be used if it have already been
    requested. This is recommended for production to avoid unnecessary file I/O.

  * `[helpers]`: Render-level helpers should be merged with (and will override)
    instance and global helper functions.

* `callback`: Function to call once the template is retrieved.

#### `renderView(viewPath, options|callback, [callback])`

Renders the template at the specified `viewPath` as the `{{{body}}}` within the
layout specified by the `defaultLayout` or `options.layout`. Rendering will use
this instance's `helpers` and partials, and passes the resulting string to the
`callback`.

This method is called by Express and is the main entry point into this Express
view engine implementation. It adds the concept of a "layout" and delegates
rendering to the `render()` method.

The `options` will be used both as the context in which the Handlebars templates
are rendered, and to signal this view engine on how it should behave, e.g.,
`options.cache=false` will load _always_ load the templates from disk.

**Parameters:**

* `viewPath`: String path to the Handlebars template file which should serve as
  the `{{{body}}}` when using a layout.

* `[options]`: Optional object which will serve as the context in which the
  Handlebars templates are rendered. It may also contain any of the following
  properties which affect this view engine's behavior:

  * `[cache]`: Whether cached templates can be used if they have already been
    requested. This is recommended for production to avoid unnecessary file I/O.

  * `[helpers]`: Render-level helpers should be merged with (and will override)
    instance and global helper functions.

  * `[layout]`: Optional string path to the Handlebars template file to be used
    as the "layout". This overrides any `defaultLayout` value. Passing a falsy
    value will render with no layout (even if a `defaultLayout` is defined).

* `callback`: Function to call once the template is retrieved.

### Statics

The following is the list of static API properties and methods provided on the
`ExpressHandlebars` constructor:

#### `getHandlebarsSemver(handlebars)`

Returns a semver-compatible version string for the specified `handlebars`
module/implementation.

This utility function is used to compute the value for an `ExpressHandlebars`
instance's `handlebarsVersion` property.


[Handlebars bug]: https://github.com/wycats/handlebars.js/pull/389


Advanced Usage Example
----------------------

As noted in the **Module Design** section, this module's implementation is
instance-based, and more advanced usages can take advantage of this. The
following example demonstrates how to use an `ExpressHandlebars` instance to
share templates with the client:

**Directory Structure:**

```
.
├── app.js
└── views
    ├── home.handlebars
    └── layouts
    │   └── main.handlebars
    └── partials
        ├── foo
        │   └── bar.handlebars
        └── title.handlebars

2 directories, 3 files
```

**app.js:**

The Express app can be implemented to expose its partials through the use of
route middleware:

```javascript
var express = require('express'),
    exphbs  = require('express3-handlebars'),

    app = express(),
    hbs;

// Create `ExpressHandlebars` instance with a default layout.
hbs = exphbs.create({
    defaultLayout: 'main'
});

// Register `hbs` as our view engine using its bound `engine()` function.
app.engine('handlebars', hbs.engine);
app.set('view engine', 'handlebars');

// Middleware to expose the app's partials when rendering the view.
function exposeTemplates(req, res, next) {
    // Uses the `ExpressHandlebars` instance to get the precompiled partials.
    hbs.loadPartials({
        cache      : app.enabled('view cache'),
        precompiled: true
    }, function (err, partials) {
        if (err) { return next(err); }

        var templates = [];

        Object.keys(partials).forEach(function (name) {
            templates.push({
                name    : name,
                template: partials[name]
            });
        });

        // Exposes the partials during view rendering.
        if (templates.length) {
            res.locals.templates = templates;
        }

        next();
    });
}

app.get('/', exposeTemplates, function (req, res, next) {
    res.render('home');
});

app.listen(3000);
```

**views/layouts/main.handlebars:**

The layout can then access these precompiled partials via the `templates` local,
and render them like this:

```handlebars
<!doctype html>
<html>
<head>
    <meta charset="utf-8" />
    <title>Example App</title>
</head>
<body>

    {{{body}}}

  {{#if templates}}
    <script src="/libs/handlebars.runtime.js"></script>
    <script>
        (function () {
            var template  = Handlebars.template,
                templates = Handlebars.templates = Handlebars.templates || {};

          {{#templates}}
            templates['{{{name}}}'] = template({{{template}}});
          {{/templates}}
        }());
    </script>
  {{/if}}

</body>
</html>
```


License
-------

This software is free to use under the Yahoo! Inc. BSD license.
See the [LICENSE file][] for license text and copyright information.


[LICENSE file]: https://github.com/ericf/express3-handlebars/blob/master/LICENSE
