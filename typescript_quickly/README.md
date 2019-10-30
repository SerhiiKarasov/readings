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
#  3.1  Working with classes 
* inheritance
```typescript
class Person {
  firstName: string;
  lastName: string;
  age: number;
  
  sayHello(): string{
    return `My name is ${this.firstName} ${this.secondName}`;
}

class Employee extends Person {
  department: string;
}

const emp = new Employee();
```
* 

Under the hood, JavaScript supports prototypal object-based inheritance, where one object can be assigned to another object as its prototype, and this happens during the runtime. During transpiling TypeScript to JavaScript, the generated code uses the syntax of the prototypal inheritance.
* if a method(s) is declared in a superclass, it will be inherited by the subclass unless the method was declared with the access qualified private
* publick methods or properties can be accessed internally by class or externally from other classes, scripts
* public is a default access modifier.
* Class members marked as protected can be accessed either from the internal class code or from class descendants.
* in javascript private, protected, publick keywords are absent, they are only for a convinience during development.
```
class Person {
    public firstName: string;
    public lastName: string;
    private age: number;
    
    protected sayHello(): string{
      return `My name is ${this.firstName} ${this.lastName}`;
  }
  
  class Employee extends Person {
    department: string;
    
    reviewPerformance(): void{
      this.sayHello();
      this.increasePay(5);
    }
    increasePay(percent: number): void{
    //   return percent*1000* 0.22 * this.age();//it will not compile as age is 
    }
  }
  
  const emp = new Employee();
  console.log(emp.increasePay(5));
```
* better way to declare class properties:
instead of   
```
class Person {
    public firstName: string;
    public lastName: string;
    private age: number;
    
    constructor(firstName: string, lastName: string, age: number){
      this.firstName = firstName;
      this.lastName = lastName;
      this.age = age;
}
```
use this one   
```
class Person {  
    constructor(public firstName: string, public lastName: string, public age: number){
}
```
* static variables(in javascript from ES6)
```
class Stat {
static stat_value = 100;

use(){
    Stat.stat_value--;
    console.log(`value is : ${Stat.stat_value}`);
    }
}
const s1 = new Stat();
s1.use();
const s2 = new Stat();
s1.use();
```
* Static class members are not shared by subclasses. 
* Singleton, constuctor is private, getInstance is static
```
class AppState {
    counter = 0;
    private static instanceRef: AppState;
    private constructor(){}
    static getInstance(): AppState{
        if (AppState.instanceRef === undefined){
            AppState.instanceRef = new AppState();
        }
        return AppState.instanceRef;
    }
}

const appState1 = AppState.getInstance();
const appState2 = AppState.getInstance();
```
* method```super()``` and keyword ```super``` need to specify from whom the method would be called(superclass or subclass)
* If both the superclass and the subclass have constructors, the one from the subclass must invoke the constructor of the superclass using the method super() as seen in listing 3.3.
```
class Person {
	constructor(public firstName: string, public lastName: string, private age: number) {}
}

class Employee extends Person {
	constructor(firstName: string, lastName: string, age: number, public department: string) {
		super(firstName, lastName, age);
	}
}
const emp = new Employee('Joe', 'Smith', 29, 'Accounting');
```
*  If a method in a subclass wants to invoke a method with the same name defined in the superclass, it needs to use keyword super instead of this when referencing the superclass method.
* abstract classes. Class that cannot be instantiated. Still class can include methods that are implemented, as well as only declared methods. 
```
abstract class Person {
	constructor(public name: string) {}

	changeAddress(newAddress: string) {
		console.log(`changeAddress ${newAddress}`);
	}
	giveDayOff() {
		console.log(`giveDayOff`);
	}

	promote(percent: number) {
		this.giveDayOff();
		this.increasePay(percent);
		console.log(`promote ${percent}`);
	}

	abstract increasePay(percent: number);
}

class Employee extends Person{
    increasePay(percent: number){
        console.log(`Increasing the salary of ${this.name} by ${percent}%`);
    }
}

class Contractor extends Person{
    increasePay(percent: number){
        console.log(`Increasing the hourly rate of ${this.name} by ${percent}%`);    }
}


const workers: Person[] = [];
workers[0] = new Employee('John');
workers[1] = new Contractor('Mary');
workers.forEach(worker => worker.promote(5));
```
* Since the descendants of Person don’t declare their own constructors, the constructor of the ancestor will be invoked automatically when we instantiate Employee and Contractor. If any of the descendants declared its own constructor, we’d have to use super() to ensure that the constructor of the Person is invoked.
* how to overload method:
    * arguments
```
class ProductService {
    getProducts();
    getProducts(id: number);
    getProducts(id?: number) {
        if (typeof id === 'number') {
          console.log(`Getting the product info for ${id}`);
        } else {
          console.log(`Getting all products`);
        }
    }
}
const prodService = new ProductService();
prodService.getProducts(123);
prodService.getProducts();
```
   * arguments and return type
```
interface Product {
  id: number;
  description: string;
}

class ProductService {

    getProducts(description: string): Product[];
    getProducts(id: number): Product;
    getProducts(product: number | string): Product[] | Product{
        if (typeof product === "number") {
          console.log(`Getting the product info for id ${product}`);
                    return { id: product, description: 'great product' };
        } else if (typeof product === "string") {
          console.log(`Getting product with description ${product}`);
          return [{ id: 123, description: 'blue jeans' },
                  { id: 789, description: 'blue jeans' }];
        } else {
           return null;
        }
    }
}

const prodService = new ProductService();

console.log(prodService.getProducts(123));

console.log(prodService.getProducts('blue jeans'));
```
* overloading constructor
```
class Product {
  id: number;
  description: string;

  constructor();
  constructor(id: number);
  constructor(id: number, description: string);
  constructor(id?: number, description?: string) {
    // Constructor implementation goes here
  }
}
```
* or similar solution using interface	
```
interface ProductProperties {
    id?: number;
    description?: string;
}

class Product {
  id: number;
  description: string;

  constructor(properties?: ProductProperties ) {
    // Constructor implementation goes here
  }
}
```
* interface example
```
interface MotorVehicle {
    startEngine(): boolean;
    stopEngine(): boolean;
    brake(): boolean;
    accelerate(speed: number);
    honk(howLong: number): void;
}
class Car implements MotorVehicle {
  startEngine(): boolean {
    return true;
  }
  stopEngine(): boolean{
    return true;
  }
  brake(): boolean {
    return true;
  }
  accelerate(speed: number) {
    console.log(`Driving faster`);
  }

  honk(howLong: number): void {
    console.log(`Beep beep yeah!`);
  }
}

const car = new Car();
car.startEngine();
```
* class need to implement each and every methods from the interface
* it is possible to set different types for the object
```
const car : Car = new Car();//if class Car has more methods than MotorVehicle, this object can invoke all of them
or 
const car : MotorVehicle = new Car();// this object can invoke only methods from the interface
```
* implementing several interfaces
```
interface Flyable {
  fly(howHigh: number);
  land();
}

interface Swimmable {
  swim(howFar: number);
}

class Car implements MotorVehicle, Flyable, Swimmable {
  // Implement all the methods from three
  // interfaces here
}
or 
class SecretServiceCar extends Car implements Flyable, Swimmable {
  // Implement all the methods from two
  // interfaces here
}
```
* extending interfaces
```
interface Flyable extends MotorVehicle{
	fly(howHigh: number);
	land();
}

class SpecialCar implements Flyable{
//to implement here all from MotorVehicle and all from Flyable
}
```
* Mantra: "Program to interfaces, not implementations"
* in TypeScript, you can declare a class that implements another class,(interface would be all  public stuff)
```
class MockProductService implements ProductService {
  // implementation goes here
}
```
* Name conventions: We named the interface IProductService starting with the capital I, while the class name was ProductService. Some people prefer using the suffix Impl with concrete implementations, e.g. ProductServiceImpl and the interface would be simply named ProductService.
* factory function may have part of business logic and return the proper instance based on conditions(tests, production)
```
function getProductService(isProduction: boolean): IProductService {
  if (isProduction) {
     return new ProductService();
  } else {
     return new MockProductService();
  }
}
const productService: IProductService;
const isProd = true;
productService = getProductService(isProd);
const products[] = productService.getProducts();
```
### Conclusion
*  You can create a class using another one as a base. We call it class inheritance.
* A subclass can use public or protected properties of a superclass.
* If a class property is declared as private, it can be used only within this class.
* You can create a class that can only instantiated once by using a private constructor.
* If a method with the same signature exists in the superclass and a subclass, we call it method overriding. Class constructors can be overridden as well. The keyword super and the method super() allow a subclass invoke the class members of a superclass.
* You can declare several signatures for a method, and this is called method overloading.
* Interfaces can declare method signatures, but can’t contain their implementations.
* You can inherit one interface from another.
* While implementing a class, see if there are certain methods that can be declared in a separate interface. Then your class has to implement that interface. This approach provide a clean way to separate declaring the functionality from implementing it.

# 4. enums
* by default starts with zero, possible to set the first value, all other would be calculated automatically
```
enum Weekdays {
  Monday = 1,
  Tuesday = 2,
  Wednesday = 3,
  Thursday = 4,
  Friday = 5,
  Saturday = 6,
  Sunday = 7
}
let dayOff = Weekdays.Tuesday;

console.log(Weekdays[3])
```
* string enums
```
enum Direction {
    Up = "UP",
    Down = "DOWN",
    Left = "LEFT",
    Right = "RIGHT",
}
```
* Redux framework(state machine) requires apps to emit state, good to be done via enums
* String enums are not reversible, and you can’t find the member’s name if you know its value.
* const enum in js is just a variable(not a function like the simple enum), hence cannot run search of member by value.

### 4.2  Generics
* TypeScript generics allow to write a function that can work with a variety of types. In other words, you can declare a function that works with a generic type(s), and the concrete type(s) can be specified later by the caller of this function.
```
let lotteryNumbers: Array<number>;
```
*  Specify the type of the array element followed by []:
```
const someValues: number[];
```
* Use a generic Array followed by the type parameter in angle brackets:
```
const someValues: Array<number>;
```
* in typescript,it is possible to add to array objects that have properties in common with the mentioned type
```
class Person {
    name: string;
}

class Employee extends Person {
    department: number;
}

class Animal {
    name: string;
    breed: string;
}

const workers: Array<Person> = [];

workers[0] = new Person();
workers[1] = new Employee();
workers[2] = new Animal();  // no errors
```
* When all elements of the array have the same type, use the syntax as in declaration of values1 - it’s just easier to read and write. But if an array can store elements of different types, you can use generics to restrict the types allowed in the array. 
```
const values1: string[] = ["Mary", "Joe"];
const values2: Array<string> = ["Mary", "Joe"];
const values4: Array<string | number> = ["Joe", 123, 567]; // no errors
```
* 4.2.2  Creating your own generic types
* programming to interfaces you think differently: "Today, I need to compare rectangles, and tomorrow they’ll ask me to compare other objects. Let me play smart and declare an interface with a function compare(). Then the class Rectangle as well as any other class in the future can implement this interface. The algorithms for comparing rectangles will be different than comparing, say, triangles, but at least they’ll have something in common and the method signature of compareTo() will look the same".
```
//bad use case, as can be used the unexpected type
interface Comparator {
  compareTo(value: any): number;
}

class Rectangle implements Comparator {

  compareTo(value: any): number {
    // the algorithm of comparing rectangles goes here
  }
}

class Triangle implements Comparator {

  compareTo(value: any): number {
    // the algorithm of comparing triangles goes here
  }
}
```

```
//good example of generic usage

interface Comparator<T> {
  compareTo(value: T): number;
}

class Rectangle implements Comparator<Rectangle>{
  compareTo(value: Rectangle): number {
    // the algorithm of comparing rectangles goes here
  }
}

class Triangle implements Comparator<Triangle>{
  compareTo(value: Triangle): number {
    // the algorithm of comparing rectangles goes here
  }
}

rectangle1.compareTo(triangle1);//typescript will report problem
rectangle1.compareTo(rectangle2); //good!!!
```
* default values of generic types
```
class A <T>{
	value: T;
}

class B extends A{//!!!!!!!!!!!transpile error
}
class C extends A<any>{
}
```
 * or
```
class A <T = any>{
	value: T;
}
//or 
class A < T = {} >{
}
class B extends A{
}
```
*  4.2.3  Creating generic functions
```
function printMe<T> (content: T): T {
    console.log(content);
    return content;
}


const printMe = <T> (content: T): T => {
    console.log(content);
    return content;
}

const a = printMe("Hello");

class Person{
  constructor(public name: string) { }
}

const b = printMe(new Person("Joe")); 
```
* comparison with generic pair
```
class Pair<K, V> {
  constructor(public key: K, public value: V) {}
}

function compare <K,V> (pair1: Pair<K,V>, pair2: Pair<K,V>): boolean {
    return pair1.key === pair2.key &&
           pair1.value === pair2.value;
}

let p1: Pair<number, string> = new Pair(1, "Apple");

let p2 = new Pair(1, "Orange");

// Comparing apples to oranges
console.log(compare<number, string>(p1, p2));

let p3 = new Pair("first", "Apple");

let p4 = new Pair("first", "Apple");

// Comparing apples to apples
console.log(compare(p3, p4));
```
* If a function can receive a function as argument or return another function, we call it a higher order function.
* higher order function that returns function with signature ```(c: number) => number```
```
(someValue: number) => (multiplier: number) => someValue * multiplier;
```
```
const outerFunct = (someValue: number) => (multiplier: number) => someValue * multiplier;
const innerFunct = outerFunct(10);
let result = innerFunc(5);
console.log(result)
```
*  
Let’s start with declaring the generic function that can take a generic type T but returns the function (c: number) ⇒ number:

```
type numFunc<T> = (arg: T) => (c: number) => number;
```
```
const noArgFunc: numFunc<void> = () => (c: number) => c + 5;
const numArgFunc: numFunc<number> = (someValue: number) => (multiplier: number) => someValue * multiplier;
const stringArgFunc: numFunc<string> = (someText: string) => (padding: number) => someText.length + padding;
const createSumString: numFunc<number> = () => (x: number) => 'Hello';
```
#  5 Decorators and advanced types
* a decorator is "a special kind of declaration that can be attached to a class declaration, method, accessor, property, or parameter. Decorators use the form @expression, where expression must evaluate to a function that will be called at runtime with information about the decorated declaration."
* Say you have class A {…} and there is a magic decorator called @Injectable() that knows how to instantiate classes and inject their instances into other objects.
```
@Injectable() class A {}
```
* Angular framework decorator(a built-in decorator @Component() for the class and @Input() for the property )
```
@Component({
	selector: 'order-processor',
	template: 'Buying {{quantity}} items'
})
export class OrderComponent{
	@Input() quantity: number;
}
```
* TypeScript doesn’t come with any built-in decorators, but you can create your own or use the ones provided by the framework or a library of your choice.
* in order to use decorators in ts:
```
--experimentalDecorators
```
```
"experimentalDecorators": true
```
### class decorator
* A class decorator is applied to the class, and the decorator function is executed when the constructor executes. The class decorator requires one parameter - a constructor function for the class.
```
function whoAmI (target: Function): void{
	console.log(`You are : \n ${target}`)
}


@whoAmI
class Friend {
	constructor (private name: string, private age: number){}
}
```
* In JavaScript, a mixin is a class that implements a certain behavior.Mixins are not meant to be used alone, but their behavior can be added to other classes. While JavaScript doesn’t support multiple inheritance, you can compose behaviors from multiple classes using mixins.
```
type constructorMixin = { new(...args: any[]): {} };

function <T extends constructorMixin> (target: T) {
   // the decorator is implemented here
}
```
* We want to create a decorator, that can accept a salutation parameter, and add to the class a new property message concatenating the given salutation and name. Also, we want to replace the code of the method sayHello() to print the message.
```
function useSalutation(salutation: string) {
  return function <T extends constructorMixin> (target: T) {
    return class extends target {
     name: string;
     private message = 'Hello ' + salutation + this.name;

     sayHello(){console.log(`${this.message}`);}
    }
  }
}

@useSalutation("Mr. ")
class Greeter {

  constructor(public name: string) { }
  sayHello() { console.log(`Hello ${this.name} `) }
}

const grt = new Greeter('Smith');
grt.sayHello();

```
### method decorator
* add logger for every function call
```
function logTrade(target, key, descriptor) {

    const originalCode = descriptor.value;

    descriptor.value = function () {

      console.log(`Invoked ${key} providing:`, arguments);
      return originalCode.apply(this, arguments);
    };

   return descriptor;
}

class Trade {
  @logTrade
  placeOrder(stockName: string, quantity: number, operation: string, traderID: number) {

     // the method implementation goes here
    }
  // other methods go here
}



const trade = new Trade();
trade.placeOrder('IBM', 100, 'Buy', 123);
```
* readonly mapped types
```
interface Person {
  name: string;
  age: number;
}

interface ReadonlyPerson {
  readonly name: string;
  readonly age: number;
}
const worker: Person = {name: "John", age: 22};

function doStuff(person: Readonly<Person>) {

    person.age = 25;
}
```
