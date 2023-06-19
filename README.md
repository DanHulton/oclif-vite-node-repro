oclif-vite-node-repro
=====================

Reproduction of [oclif](https://oclif.io/)/[vite-node](https://www.npmjs.com/package/vite-node) incompatibility.

## Setup

This is a bare installation of oclif, converted to run using vite-node.  The changes from the base oclif installation are as follows:

- vite-node is installed via npm.
- The "dev" command's shebang is modified to use vite-node
- The [oclif ESM Guide](https://oclif.io/docs/esm) is followed, as otherwise you get `SyntaxError: Cannot use import statement outside a module` errors:
  - `tsconfig.json` is modified, per the guide.
  - The `dev` command is renamed to `dev.js` and the command code is replaced with the version from the guide.
  - (Not listed in the guide, but still necessary), `"type": "module"` is added to package.json.

## The error

Running `DEBUG=* ./bin/dev.js` returns the following error:

```
(node:51441) [ERR_UNKNOWN_FILE_EXTENSION] TypeError Plugin: oclif-vite-node-repro [ERR_UNKNOWN_FILE_EXTENSION]: Unknown file extension ".ts" for /Users/danhulton/dev/oclif-vite-node-repro/src/commands/hello/index.ts
module: @oclif/core@2.8.7
task: toCached
plugin: oclif-vite-node-repro
root: /Users/danhulton/dev/oclif-vite-node-repro
See more details with DEBUG=*
(Use `node --trace-warnings ...` to show where the warning was created)
TypeError Plugin: oclif-vite-node-repro [ERR_UNKNOWN_FILE_EXTENSION]: Unknown file extension ".ts" for /Users/danhulton/dev/oclif-vite-node-repro/src/commands/hello/index.ts
    at new NodeError (node:internal/errors:393:5)
    at Object.getFileProtocolModuleFormat [as file:] (node:internal/modules/esm/get_format:79:11)
    at defaultGetFormat (node:internal/modules/esm/get_format:121:38)
    at defaultLoad (node:internal/modules/esm/load:81:20)
    at nextLoad (node:internal/modules/esm/loader:163:28)
    at ESMLoader.load (node:internal/modules/esm/loader:605:26)
    at ESMLoader.moduleProvider (node:internal/modules/esm/loader:457:22)
    at new ModuleJob (node:internal/modules/esm/module_job:63:26)
    at ESMLoader.#createModuleJob (node:internal/modules/esm/loader:480:17)
    at ESMLoader.getModuleJob (node:internal/modules/esm/loader:434:34)
module: @oclif/core@2.8.7
task: toCached
plugin: oclif-vite-node-repro
root: /Users/danhulton/dev/oclif-vite-node-repro
See more details with DEBUG=*
```

## Investigation

The actual line that causes the error is [line 86 in @oclif/core/src/module-loader.ts](https://github.com/oclif/core/blob/main/src/module-loader.ts#LL86C7-L86C99).  `isESM` from the previous line is set to `true`, so it is attempting to run `_importDynamic`, which is just a function that runs `return import(modulePath)`.  This can effectively be replaced with `const module = await import(url.pathToFileURL(filePath))`, and indeed if so done, returns the same error.  This means the issue is not with the clever transpile-avoiding require function, but with the import function at all.

My suspicion is that vite-node is not being used to import files at this point, whether because the vite-node loader isn't used to load anything originating from `node-modules` or because it's not used in `.js` files (as the oclif module is compiled to javascript on publish), or for some other reason I do notknow. 
