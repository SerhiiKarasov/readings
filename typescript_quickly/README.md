# https://livebook.manning.com/book/typescript-quickly



# 1 Getting familiar with TypeScript
* typescript is transplited to javascript
* typescript is superset of ecma script(aka javascript), i.e. .js can be renamed to .ts(if there is no problem related to dynamic typization all should be ok)
* official page:  www.typescriptlang.org. 
* online typescript compliler to js: http://www.typescriptlang.org/play/index.html
* types in TypeScript are optional
* ts allows to use the latest ECMAScript syntax (e.g. async and await)
* list of languages that compiles to js: https://github.com/jashkenas/coffeescript/wiki/list-of-languages-that-compile-to-js
* TypeScript follows the latest specifications of ECMAScript and adds to them types, interfaces, decorators, class member variables (fields), generics, enums, the keywords public, protected, and private and more.
* TS roadmap: https://github.com/Microsoft/TypeScript/wiki/Roadmap
## deploying app written in ts
* Ts files (a.ts, b.ts) -> complied > Js files(a.js, b.js) -> bundle -> Js file(main.js) ->deploy -> Js engine(main.js)
* Bundlers like Webpack or Rollup, only concatenate multiple JavaScript files, but can optimize the code and remove unused code (a.k.a. perform tree-shaking). 
* js code may not be rewritten to ts,e definition files with the name extension .d.ts can be used for type checking

## Using TypeScript compiler
```
npm -v
npm install -g typescript
tse -v
```
* create a file
```
function getFinalPrice(price: number, discount: number) {
  return price - price/discount;
}
console.log(getFinalPrice(100, 10));
console.log(getFinalPrice(100, "10%"));
```
* run this code, note that js code would be generated even even with error being printed
```
tsc main
```
* or run with compile option that will block generation of .js file
```
tsc main --noEmitOnError true
```
* you can use --t option to choose JS syntax
```
tsc --t ES5 main
```
* it is possible to preconfigure the compilation(source, target, directories), need to create the file tsconfiguration.json
```javascript
{
  "compilerOptions": {
      "baseUrl": "src",
      "outDir": "./dist",
      "noEmitOnError": true,
      "target": "es5"
  }
}
```
* to start project(actually to create tsconfiguration.json file)
```
tsc --init
```
* in tsconfiguration.json it is possible to inherit configuration using extend control word
* REPL(Read-Evaluate-Print-Loop) executes your code in interactive console mode
* Good to install in vscode
TSLint - integrated the TypeScript linter,
Prettier - code formatter,
Path Intellisense - auto-completes file paths.
* online ide based on vscode - https://stackblitz.com/



#  2 Basic and custom types
* The type can be assigned to a variable either explicitly by a software developer or implicitly (a.k.a. inferred types) 
## 2.1.1  Basic type annotations
* add a colon and a type annotation to specify the variable type
```
let firstName: string;
let age: number;
```
## Types:
* string - for textual data
* boolean - for true/false values
* number - for numeric values
* symbol - unique value created by calling the Symbol constructor
* any - for variables that can hold values of various types, which may be unknown when you write code
* unknown - a counterpart of any but no operations are permitted on an unknown without first asserting or narrowing to a more specific type.
* never - for representing an unreachable code (we’ll provide an example shortly)
* void - an absence of a value
* Symbol - immutable, and unique. Sym1 is not eq sym2
```
const sym1 = Symbol("orderID");
const sym2 = Symbol("orderID");
```
```
const ord = Symbol('orderID');

const myOrder = {
    ord: "123"
};

console.log(myOrder['ord']);
```
* Typescript also has null and undefined(not assigned or function that do not return value) values
* function that returns string or null
```
function getName(): string | null
{

};
```
* any type, to this variable can assign numeric, textual, boolean or custom type, but that is not an optimal behaviour
* never type - assigned to function that never returns or just throws an error
```
const logger = () =>
{
  ...
};
```
