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
* in tsconfiguration.json it is possible to inherit configuration using ```extends``` control word
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

* typescript may specify type from the 'rvalue', hence type specification is redundant here
```typescript
let name1: string = 'something';
```
* it is possible to infer type/value from the literal. The variable name3 will only allow one value: John Smith and an attempt to assign another value to a variable of a literal type would result in type checker’s error:
```typescript
let name3: 'Name';
name3 = 'Mary Lou';  // error: Type '"Mary Lou"' is not assignable to type '"John Smith"'
```

* if declare variable without initialization -> typescript will infer its type as ```any```. In javascript it would such uninitialized value is *undefined*
* infering of the type is called *type widening*
* Type annotations are used not only for declaring variable types, but also for declaring types of function arguments and their return values,
### Types in function declarations
* example of functions in javascript vs typescript
```javascript
//javascript
function calcTax(state, income, dependents){
  if(state = "NY"){
    return income * 0.006 - dependents * 500;
  }
  else if(state == 'NJ'){
    return income * 0.006 - dependents * 300;
  }
}
```

```typescript
//typescript 
function calcTax(state: string, income: number, dependents: number) : number{
  if(state = "NY"){
    return income * 0.006 - dependents * 500;
  }
  else if(state == 'NJ'){
    return income * 0.006 - dependents * 300;
  }
}
```
### the union type
* allow a value to be of several types
```
let padding: string | number;
```
* check built-in type of the variable:
```
    if (typeof padding === "number") {
        return Array(padding + 1).join(" ") + value;
    }
```
* if the type is a custom one, need to use ```instanceof```
```
if (person instanceof Person) {...}
```
### define custome type
* use keyword ```type``` to declare class or an interface(or use ```enum```)
```
type Foot = number;
type Pound = number;


type Patient = {
  name: string;
  height: Foot;
  weight: Pound;
}


let patient: Patient = {
  name: 'Joe Smith',
  height: 5,
  weight: 100
}
```
* function examples
```
function myAdd(x: number, y: number): number {  return x + y; }
let myAdd                                   = function(x: number, y: number): number { return x + y; };
let myAdd: (x: number, y: number) => number = function(x: number, y: number): number { return x + y; };

```
* if during initialization to forget about one of properties(e.g. weight) typescript will complain that initialized type is not the one we are creating, some property is missing
* it is possible to create optional properties
```
type Patient = {
        name: string;
        height: Height;
        weight?: Weight;
}

let patient: Patient = {
        name: 'Joe Smith',
        height: 5
}
```

* the type keyword for declaring a type alias to a function signature, in this case an object can have properties of any type:
```
type ValidatorFn = (c: FormControl) => { [key: string]: any }| null
```
function accepts object of FormControl, and returns either an object describing error or null.  
```
class FormControl {
constructor (initialValue: string, validator: ValidatorFn | null) {...}
}
```
* typescripts can declare classes, javascript cannot declare properties in class
```
//typescript
class Person{
  firstName: string;
  lastName: string;
  age: number;
}


//javascript
"use strict"
class Person{
}

//initialization is same in both cases
const p = new Person();
p.firstName = "john";
p.lastName = "Smith";
p.age = 25;
```
* class constuctors(allows set all properties in one line). There are level qualifiers ```public``, ```private```, ```protected```
```typescript
//typescript
class Person{
  constructor(public firstName: string, 
              public lastName: string,
              public age: nummber) {};
}              
//javascript ES6
class Person{
  constructor(firstName, lastName, age) {
  this.firstName = firstName;
  this.lastName = lastName;
  this.age = age;
};
};

//initialization is the same in both cases
const p = new Person("A", "B", 25);
//possible but stupid, why to explicitly set type to Person if that is the const and we know the type we are setting here :)
const p: Person = new Person("John", "Smith", 25);
```
* possible to mark properties of the class as readonly(can be initialized at declaration, or in class constructor). const cannot be used with class properties
```typescript
class Block {
  readonly nonce: number;
  readonly hash: string;

  constructor (
    readonly index: number,
    readonly previousHash: string,
    readonly timestamp: number,
    readonly data: string
  ) {
    const { nonce, hash } = this.mine();
    this.nonce = nonce;
    this.hash = hash;
  }
  // The rest of the code is omitted for brevity
}
```
### interfaces
*  JavaScript doesn’t support interfaces but TypeScript does.
* TypeScript includes the keywords ```interface``` and ```implements```
```typescript
interface Person{
  firstName: string;
  lastName: string;
  age: number;
}
```
* if declare class with ```class``` it's instance van be used as a value or be instantiated via ```new```. While ```interface``` class is not an object, it would be just used by tpscript compiler checks, and would not be present in javascript code. 
* if two types has the same members, they are compatible. as typescript has structural type system(not nominal)
```
class Person {
        name: string;
}

class Customer {
        name: string;
}

const cust: Customer = new Person();
```
of course access modifiers can influence this
* what to choose ```type```, ```class```, ```interface```: if type doesn't need to be used for instantiating objects -> user interface, otherwirse class. For type safety use interface or type(neither is available in javascript), Interfaces cannot be used in unions or intersections(types can be used here)
* assignable only that class that has more or same properties
* union of custom types
```
export class SearchAction {
  actionType = "SEARCH";

  constructor(readonly payload: {searchQuery: string}) {}
}

export class SearchSuccessAction {
  actionType = "SEARCH_SUCCESS";

  constructor(public payload: {searchResults: string[]}) {}
}

export class SearchFailedAction {
  actionType = "SEARCH_FAILED";
}

export type SearchActions = SearchAction | SearchSuccessAction | SearchFailedAction;
```
* discriminated union, in this case it is ```Shape```
```
interface Rectangle {
    kind: "rectangle";
    width: number;
    height: number;
}
interface Circle {
    kind: "circle";
    radius: number;
}

type Shape = Rectangle | Circle;
function area(shape: Shape): number {
  switch (shape.kind) {
    case "rectangle": return shape.height * shape.width;
    case "circle": return Math.PI * shape.radius ** 2;
  }
}

const myRectangle: Rectangle = { kind: "rectangle", width: 10, height: 20 };
console.log(`Rectangle's area is ${area(myRectangle)}`);

const myCircle: Circle = { kind: "circle", radius: 10};
console.log(`Circle's area is ${area(myCircle)}`);
```
* the ```in``` guard, acts as a narrowing expression for types.
```
interface A { a: number };
interface B { b: string };

function foo(x: A | B) {
    if ("a" in x) {
        return x.a;
    }
    return x.b;
}
```
* type ```unknown``` was introduced in TypeScript 3.0. If you declare a variable of type unknown, the compiler will force you to narrow its type down before accessing its properties. 
* diff any vs unknown
```
type Person = {
  address: string;
}
let person1: any;
person1 = JSON.parse('{ "adress": "25 Broadway" }');
console.log(person1.address);  //misspelled address, will print undefined
```
```
let person2: unknown;
person2 = JSON.parse('{ "adress": "25 Broadway" }');
console.log(person2.address);//compilation failure, as misspelled.
```
* custom guards
```
const isPerson = (object: any): object is Person => "address" in object;

if (isPerson(person2)) {
    console.log(person2.address);
} else {
    console.log("person2 is not a Person");
}

const isPerson = (object: any): object is Person => !!object && "address" in object; // better as checks if object is truthy
```
