# Ember CLI HTMLBars

<a href="https://github.com/ember-cli/ember-cli-htmlbars/actions"><img alt="Build Status" src="https://github.com/ember-cli/ember-cli-htmlbars/workflows/CI/badge.svg"></a>

## Compatibility

* Ember.js v3.8 or above
* Ember CLI v3.8 or above
* Node.js v12 or above

## Tagged Template Usage / Migrating from `htmlbars-inline-precompile`

Starting with version 4.0, this addon now includes the testing helper from [ember-cli-htmlbars-inline-precompile](https://github.com/ember-cli/ember-cli-htmlbars-inline-precompile)

This will require an update to the imports of the `hbs` helper in your tests:

Prior syntax:

```
import hbs from 'htmlbars-inline-precompile';

...

await render(hbs`
  <MyComponent />
`);
```

New syntax:

```
import { hbs } from 'ember-cli-htmlbars';

...

await render(hbs`
  <MyComponent />
`);
```

There is a [codemod](https://github.com/ember-codemods/ember-cli-htmlbars-inline-precompile-codemod) available to automate this change.

## Additional Trees

For addons which want additional customizations, they are able to interact with
this addon directly.

```ts
interface EmberCLIHTMLBars {
  /**
    Supports easier transpilation of non-standard input paths (e.g. to transpile
    a non-addon NPM dependency) while still leveraging the logic within
    ember-cli-htmlbars for transpiling (e.g. custom AST transforms, colocation, etc).
  */
  transpileTree(inputTree: BroccoliTree): BroccoliTree;
}
```

### `transpileTree` usage

```js
// find the ember-cli-htmlbars addon
let htmlbarsAddon = this.addons.find(addon => addon.name === 'ember-cli-htmlbars');

// invoke .transpileTree passing in the custom input tree
let transpiledCustomTree = htmlbarsAddon.transpileTree(someCustomTree);
```

## Adding Custom Plugins

You can add custom plugins to be used during transpilation of the `addon/` or
`addon-test-support/` trees of your addon (or the `app/` and `tests/` trees of an application)
by registering a custom AST transform.

```js
var SomeTransform = require('./some-path/transform');

module.exports = {
  name: 'my-addon-name',

  included: function() {
    // we have to wrap these in an object so the ember-cli
    // registry doesn't try to call `new` on them (new is actually
    // called within htmlbars when compiling a given template).
    this.app.registry.add('htmlbars-ast-plugin', {
      name: 'some-transform',
      plugin: SomeTransform
    });
    
    this._super.included.apply(this, arguments);
  }
};
```

### Options for registering a plugin

* `name` - String. The name of the AST transform for debugging purposes.
* `plugin` - A function of type [`ASTPluginBuilder`](https://github.com/glimmerjs/glimmer-vm/blob/master/packages/%40glimmer/syntax/lib/parser/tokenizer-event-handlers.ts#L329-L341).
* `dependencyInvalidation` - Boolean. A flag that indicates the AST Plugin may, on a per-template basis, depend on other files that affect its output.
* `cacheKey` - function that returns any JSON-compatible value - The value returned is used to invalidate the persistent cache across restarts, usually in the case of a dependency or configuration change.
* `baseDir` - `() => string`. A function that returns the directory on disk of the npm module for the plugin. If provided, a basic cache invalidation is performed if any of the dependencies change (e.g. due to a npm install/upgrade).

### Implementing Dependency Invalidation in an AST Plugin

Plugins that set the `dependencyInvalidation` option to `true` can provide function for the `plugin` of type `ASTDependencyPlugin` as given below.

Note: the `plugin` function is invoked without a value for `this` in context.

```ts
import {ASTPluginBuilder, ASTPlugin} from "@glimmer/syntax/dist/types/lib/parser/tokenizer-event-handlers";

export type ASTDependencyPlugin = ASTPluginWithDepsBuilder | ASTPluginBuilderWithDeps;

export interface ASTPluginWithDepsBuilder {
  (env: ASTPluginEnvironment): ASTPluginWithDeps;
}

export interface ASTPluginBuilderWithDeps extends ASTPluginBuilder {
  /**
   * @see {ASTPluginWithDeps.dependencies} below.
   **/
  dependencies(relativePath): string[];
}

export interface ASTPluginWithDeps extends ASTPlugin {
  /**
   * If this method exists, it is called with the relative path to the current
   * file just before processing starts. Use this method to reset the
   * dependency tracking state associated with the file.
   */
  resetDependencies?(relativePath: string): void;
  /**
   * This method is called just as the template finishes being processed.
   *
   * @param relativePath {string} A relative path to the file that may have dependencies.
   * @return {string[]} paths to files that are a dependency for the given
   * file. Any relative paths returned by this method are taken to be relative
   * to the file that was processed.
   */
  dependencies(relativePath: string): string[];
}
```

## Precompile HTMLBars template strings within other addons

```javascript
module.exports = {
  name: 'my-addon-name',

  setupPreprocessorRegistry: function(type, registry) {
    var htmlbarsPlugin = registry.load('template').find(function(plugin) {
      return plugin.name === 'ember-cli-htmlbars';
    });

    // precompile any htmlbars template string via the precompile method on the
    // ember-cli-htmlbars plugin wrapper; `precompiled` will be a string of the
    // form:
    //
    //   Ember.HTMLBars.template(function() {...})
    //
    var precompiled = htmlbarsPlugin.precompile("{{my-component}}");
  }
};
```

### Custom Template Compiler

You can still provide a custom path to the template compiler (e.g. to test
custom template compiler tweaks in an application) by:

```js
// ember-cli-build.js

module.exports = function(defaults) {
  let app = new EmberApp(defaults, {
    'ember-cli-htmlbars': {
      templateCompilerPath: `some_path/to/ember-template-compiler.js`,
    }
  });
};
```

## Using as a Broccoli Plugin

```javascript
var HtmlbarsCompiler = require('ember-cli-htmlbars');

var templateTree = new HtmlbarsCompiler('app/templates', {
  isHTMLBars: true,

  // provide the templateCompiler that is paired with your Ember version
  templateCompiler: require('./bower_components/ember/ember-template-compiler')
});
```
