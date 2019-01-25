# Support for ESM syntax in “secondary” entry points in Node.js

**Contributors**: Geoffrey Booth (@GeoffreyBooth), Guy Bedford (@guybedford), John-David Dalton (@jdalton), Jan Krems (@jkrems), Saleh Abdel Motaal (@SMotaal)

## Overview

This proposal aims to define ways that Node can support ES module syntax for the following “secondary” entry point types:

- `--eval`, e.g. `node --eval 'console.log("hello")'`

- `STDIN`, e.g. `echo 'console.log("hello")' | node`

- extensionless files, e.g. `/usr/local/bin/npm`

The “primary” entry point type is direct Node execution, e.g. `node file.js`. This is covered by the [File Specifier Resolution proposal](https://github.com/GeoffreyBooth/node-import-file-specifier-resolution-proposal/), which this proposal extends.

## Motivating Examples

- `--eval` is commonly used by users for debugging and experimentation, similar to using the REPL.

- Tools like Babel and CoffeeScript send their transpiled JavaScript output to Node to execute via `--eval` or `STDIN`. For example, `coffee 'console.log "hello"'` gets transpiled into `console.log("hello")` which is then passed on to Node via `--eval` for execution.

- Tools like NPM, Babel, CoffeeScript, Gulp and others that install globally often put an extensionless executable file in a folder in the user’s `$PATH`, such as `/usr/local/bin/npm`.

## Proposal: `--module` command line flag

Node will support a new command line flag, `--module` and `-m`. This tells Node to treat the initial entry point as ESM.

### Examples

The following would each print your system’s file path separator (`/` or `\`):

```bash
node --module --eval 'import { sep } from "path"; console.log(sep)'

echo 'import { sep } from "path"; console.log(sep)' | node --module

NODE_OPTIONS='--module' node --eval 'import { sep } from "path"; console.log(sep)'

export NODE_OPTIONS='--module';
node --eval 'import { sep } from "path"; console.log(sep)'
```

The following would throw an error:

```bash
node --module --eval 'require("path")'
```

### Note on Package Scope and `--eval` and `STDIN`

The [initial entry point package scope](https://github.com/GeoffreyBooth/node-import-file-specifier-resolution-proposal/#initial-entry-point) parse goal does _not_ extend to `--eval` or `STDIN` input that may be executed from a working folder that’s within an ESM package scope. These “file-less” forms of input are always treated as CommonJS unless the `--module` flag is also present, or set via `NODE_OPTIONS`.

## Extensionless Files

Extensionless files are typically executables like `/usr/local/bin/npm` or `./node_modules/.bin/babel`. They are typically JavaScript files with a first line of `#!/usr/bin/env node`.

### Extensionless Files Within a Package Scope

Per the [File Specifier Resolution proposal](https://github.com/GeoffreyBooth/node-import-file-specifier-resolution-proposal/), `node file.js` treats `file.js` as ESM if it is in an ESM _package scope._ A file is in an ESM package scope if it is in or under a folder with a `package.json` that contains an ESM signifying field. Extensionless files executed by Node within an ESM package scope would inherit that ESM scope just as `file.js` does.

This package scope concept covers cases of extensionless files within a package, such as `node_modules/typescript/bin/tsc`.

### Extensionless Files Outside Any Package Scope

There is one use case for extensionless files outside of a package scope: globally-installed command-line tools, such as NPM. On MacOS, for example, the `npm` command executes `/usr/local/bin/npm`. (That happens to be a symlink to `/usr/local/node_modules/npm/bin/npm-cli.js`, but when executing `npm` Node’s `process.argv[1]` is `/usr/local/bin/npm`, the source of the symlink and not its target.) Most users won’t have a `package.json` in their `/usr/local/bin` folder or any of its parent folders, and so `/usr/local/bin/npm` will execute as CommonJS unless `--module` is set in the `NODE_OPTIONS` environment variable.

Since it’s not possible to add `--module` to the `#!/usr/bin/env node` line in a cross-platform compatible way, we are opting to leave this case as CommonJS only. Authors of such tools can “opt in” to ESM by making the initial entry point CommonJS file a bridge to an ESM package, for example:

```js
#!/usr/bin/env node
try { // Try to load ESM version of the package
  new Function("import('./esm/index.js').catch(exception => console.error(exception))")();
} catch (exception) { // Fallback to the CommonJS version
  require('./commonjs/index.js');
}
```

As this use case is limited, we consider this solution good enough until Node eventually changes its “outside of package scope” default from CommonJS to ESM.
