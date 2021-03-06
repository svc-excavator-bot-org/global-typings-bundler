[![No Maintenance Intended](http://unmaintained.tech/badge.svg)](http://unmaintained.tech/)

[![npm version](https://badge.fury.io/js/global-typings-bundler.svg)](https://badge.fury.io/js/global-typings-bundler)
[![Circle CI](https://circleci.com/gh/palantir/global-typings-bundler.svg?style=svg&circle-token=7aa0422260d471482bcbc9719d609e530f32ccda)](https://circleci.com/gh/palantir/global-typings-bundler)

## global-typings-bundler 

Bundles your TypeScript definition files, like a JS module bundler does for your source files.

### Important: Deprecation Notice

_8/25/16_: This project is now deprecated in favor of built-in support for UMD module definitions in TypeScript 2.0: https://github.com/Microsoft/TypeScript/wiki/What%27s-new-in-TypeScript#support-for-umd-module-definitions

---

**WARNING**: experimental/unstable, use at your own risk

* [Usage](#usage)
* [Motivation](#motivation)
* [Caveats](#caveats)
* [Development](#development)
* [Distribution](#distribution)

### Usage

```sh
npm install global-typings-bundler --save-dev
```

#### Example

```ts
import { writeFileSync } from "fs";
import { bundleTypings } from "global-typings-bundler";

const result = bundleTypings("MyReactComponent", "path/to/entry/index.d.ts", {
    "react": "React",
    "react-dom": "ReactDOM",
    "react-addons-css-transition-group": "React.addons.CSSTransitionGroup",
});

writeFileSync("my-global-typings.d.ts", result, "utf8");
```

#### API

##### `bundleTypings(globalName: string, typingsEntryPoint: string, externals?: Object): string`

- `globalName`: The name of the desired global namespace.
- `typingsEntryPoint`: Path to the typings file for the module's entry point.
- `externals`: An optional object which maps external module names to their corresponding globals.
  For example, "react" would map to "React", "react-addons-test-utils" -> "React.addons.TestUtils", etc.
  __If your module imports any external libraries, you must include them here or the bundling will fail__.

#### Input

Granular external module definition files as generated with `tsc --module commonjs`:

```
my-module/
├── foo.d.ts
├── bar.d.ts
├── index.d.ts
└── someFolder/
    └── nestedModule.d.ts
```

```ts
// index.d.ts
export { IFoo, parseExport } from "./foo";
export { IBar } from "./bar";
```

```ts
// foo.d.ts
import * as ts from "typescript";
import { someGlobalVariable } from "./someFolder/nestedModule";
export interface IFoo {
    ...
}
export function parseExport(exportDecl: ts.ExportDeclaration): IFoo[] {
    ...
}
```

```ts
// someFolder/nestedModule.d.ts
export const someGlobalVariable: string;
```

#### Output

A flattened `.d.ts` file that matches the shape of the namespaces created by a JS bundler like webpack or browserify:

```ts
declare namespace __MyModule.__SomeFolder.__NestedModule {
    export const someGlobalVariable: string;
}
declare namespace __MyModule.__Foo {
    import someGlobalVariable = __MyModule.__SomeFolder.__NestedModule.someGlobalVariable;
    export interface IFoo {
        ...
    }
    export declare function parseExport(exportDecl: ts.ExportDeclaration): IFoo[];
}
declare namespace MyModule {
    export import IFoo = __MyModule.__Foo.IFoo;
    export import parseExport = __MyModule.__Foo.parseExport;
    export import IBar = __MyModule.__Bar.IBar;
}
```

The `__` namespaces are fake and do not correspond to real values at runtime in JS.

### Motivation

As we transitioned our TypeScript libraries to ES6 module syntax and our applications to use a JS module loader,
we wanted to retain interoperability with older applications that did not use a module loader or bundler. This
is easy to do for the JS -- you simply bundle using a tool like webpack and you're ready to use the browser-global
version of the library with a package manager like Bower. However, the typings generated by the compiler are unusable
in these legacy applications because of the lack of module loader (they can't have any external module imports/exports).
So, we built this tool to bridge the gap. It allows you to author a TypeScript library in ES6 module syntax and
distribute it as a strongly typed CommonJS module on NPM as well as a strongly typed global module on Bower.

Note that this tool is distinct from [dts-bundle](https://github.com/TypeStrong/dts-bundle), which also bundles up
external module definition files, but does not flatten the module structure into namespaces.

### Caveats

The following structures are not currently supported and will cause the library to either throw an error or generate incorrect typings.

* default exports: `export default ...`
* default imports mixed with named imports: `import foo, { bar } from ...`
* wildcard re-exports: `export * from ...`
* named export statements without a from clause: `export { foo, bar as baz }`. (Note: `export { foo, bar as baz } from ...` _is_ supported.)

### Development

Quick Start:

```
npm run lint
npm run build
```

#### Testing

In order to test your changes:

* Add/modify test cases as necessary in `test/cases`. Each test case is its own directory
  and has a file named `params.json` which supplies the parameters to pass when building the bundled typings file.
  If your code changes are supposed to affect the output of the library, ensure at least one test case is affected
  as well. If you're simply refactoring code, it's fine to leave the test cases as they are.
* Run `npm run build` to build the latest version of your code.
* Run `npm run test` to generate output for all test cases into the `test/output` directory.
    * If the command succeeds, your code changes didn't affect the output from any test cases.
    * If the command fails, your code changes have affected the output from at least one test case.
      Use `npm run test-diff` to see the differences or view the differences between
      `test/accepted-output` and `test/output` with your favorite editor/diff tool.
* If the output in `test/output` is what is desired, run `npm run test-accept`.
* When you commit your code changes, also commit the changes to `test/accepted-output`.


### Distribution

Publishing to NPM:

```
npm run all
npm publish dist/
```
