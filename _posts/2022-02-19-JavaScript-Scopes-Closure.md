---
layout: post
title: JavaScript | Scopes, Closure
published: true
tags: JavaScript
---

JavaScript-ში დამწყებებისთვის ერთ-ერთი ყველაზე დამაბნეველი თემა Scope (სეგმენტი) და Clouse-რე არის. ამ სტატიაში ვეცდები მარტივად ავხსნა თუ რას ნიშნავს ეს ტერმინები.

### Scope
JavaScript-ში Scope გვაქვს - ლექსიკური (Lexical Scope), ფუნქციონალური (Functional Scope), მოდულარული (Module Scope), ბლოკის (Block Scope) და გლობალური (Global Scope) სეგმენტები, ასევე არსებობს დინამიური სეგმენტი (Dynamic Scope) რომელიც JavaScript-ში არ არის. 

სეგმენტი შეგვიძლია როგორც კონტეინერი წარმოვიდგინოთ ცვლადისთვის სადაც მისი სიცოცლხის დრო და ხილვადობა განისაზღვრება სხვა ფუნქციებისათვის ან ცვლადებისთვის.

### Global Scope

ცვლადები/ფუნქციები რომლებიც გამოცხადებულია ფუნქციის, ბლოკის ან მოდულის გარეთ მიეკუთვნებიან გლობალურ სეგმენტს, ისინი კოდში ნებისმიერ ადგილას არიან ხელმისაწვდომები.

ბრაუზერისთვის გლობალური ობიექტია `window` ხოლო `node`-სთვის `global`. 

`index.js`
```js
function globalFunction() {
    console.log(this)
}

window.globalFunction()
```
კოდის შედეგი იქნება:
```
Window {window: Window, self: Window, document: document, name: '', location: Location, …}
```

რადგან `globalFunction` გლობალურ სეგმენტშია გამოცხადებული ის `window` ობიექტს მიეკუთვნება.


### Module Scope
მოდულში გამოცხადებული ცვლადები მხოლოდ ამ მოდულისთვის იქნება ხელმისაწვდომი, თუ გვინდა რომ ცვლადი სხვა მოდულში მივიღოთ მაშინ ეს ცვლადი უნდა დავაექსპორტოთ.

`a.js`
```js
const a = 1;
export { a };
```

და `import`-ით მივიღოთ ცვლადი სხვა მოდულში.

`b.js`
```js
import { a } from './a.js';
```

რადგან მოდული იმპორტის დროს შექმნის ფუნქციას სადაც მოათავსებს მთელ კოდს და განუსაზღვრავს სეგმენტს. მაგალითად NodeJS-სი მოდულს ასე გარდაქმნის:

`index.js`
```js
console.log(arguments);
```
იმპორტის დროს გარდაიქმნება:
```js
funtion(exports, module, require, __filename, __dirname) {
  console.log(arguments);
  return module.exports
}
```

### Function Scope
ფუნქციონალურ სეგმენტში გამოცხადებული ცვლადები და ფუნქციის პარამეტრები მხოლოდ ამ ფუნქციის შიგნით იქნება ხელმისაწვდომი.

```js
function foo() {
    const variable = 1;
}

console.log(variable) // ReferenceError: variable is not defined
```

#### var vs let and const

ცვლადებს, რომლებიც გამოცხადებულია `var`-ით, მხოლოდ ფუნქციური სეგმენტი გააჩნიათ და კომპილაციის დროს ფუნქციის თავში თავსდებიან.

```js
function foo() {
    console.log(x) //undefined
    var x = 1;
}
```

დაიბეჭდება `undefined` რადგან var-ით გამოცხადებული ცვლადი ფუნქციის თავში ინაცვლებს კომპილაციისას, რასაც `Hoisting` ეწოდება.

```js
function foo() {
    var x = undefined;
    console.log(x) //undefined
    x = 1;
}
```

```
Hoisting არის JavaScript-ის მექანიზმი სადაც გამოცხადებული ცვლადები და ფუნქციები მისი სეგმენტის თავში თავსდებიან.
```

თუ ცვლადი გამოცხადებულია `var`-ით, ფუნქცისს გარეთ მისი მნიშვნელობა `global` ობიექტში ჩაიწერება.

```js
var foo = 1;
window.foo // 1
```

`let` და `const`-ით გამოცხადებული ცვლადების სეგმენტი არის ბლოკი (Block Scope) თუ შევქმნით ცვლადს ფუნქციების გარეთ, მათი მნიშვნელობა `global` ობიექტში არ ჩაიწერება.

```js
console.log(foo) //ReferenceError: foo is not defined
let foo = 1;
```

ასევე `var` შეგვიძლია გადავადეკლარიროთ `let/const` განსხვავებით.

```js
var x = 1;
var x = 2;
console.log(x) // 2

let y = 1;
let y = 2; // SyntaxError: Identifier 'y' has already been declared 
```

### Block Scope

ბლოკის სეგმენტი იქმნება ფიგურულ ბრჭყალებს შორის `{ }`. როგორც ავღნიშნეთ `let` და `const`-ით გამოცხადებულ ცვლადებს გააჩნიათ ბლოკის სეგმენტი და ისინი მხოლოდ იმ ბლოკში არიან ხელმისაწვდომები სადაც გამოცადდნენ.

```js
let x = 1;
{
  let x = 2;
}
console.log(x) // 1
```

```js
var x = 1;
{
  var x = 2;
}
console.log(x) // 2
```

ავიღოთ მაგალითსთვის შემდეგი კოდი:
```js
function loop(){
    for(var i=0; i<5; i++){
        setTimeout(function logValue(){
            console.log(i);         //5
        }, 100);
    }
};

loop();
```

ეს კოდი `console`-ში დაბეჭდავს `5`-ს ყოველ იტერაციაზე. რადგან `setTimeout`-ი 100 მილიწამიანი დაგვიანებით სრულდება, მისი შესრულების დროს `i`-ცვლადის მნიშვნელობა უკვე `5` იქნება.

```js
function loop(){
    for(let i=0; i<5; i++){
        setTimeout(function logValue(){
            console.log(i);         // 0, 1, 2, 3, 4
        }, 100);
    }
};

loop();
```

ხოლო `let` ყოველ ჯერზე ახალ ლოკალურ ცვლადს შექმნის, ამიტომ დაიბეჭდება `0, 1, 2, 3, 4`

### Lexical Scope
ლექსიკურ სეგმენტს ასევე უწოდებენ სტატიკურ სეგმენტს ის ცვლადისთვის განსაზღვრავს ადგილს (კონტეინერს) სადაც ის შეიქმნა. მაგალითად თუ ცვლადი შეიქმნა გლობალურ სეგმენტში მიდი ლექსიკური სეგმენტი იქნება გლობალური, თუ ცვლადი შეიქმნა ფუნქციაში მისი ლექსიგური სეგმენტი იქნება ის ფუნქცია სადაც ცვლადი გამოცხადდა.

```js
function foo() { 
  console.log( a ); 
}


function bar() { 
  var a= 3;
  foo(); 
}


var a=2; 


bar();
```

რა იქნება `a` ცვლადის მნიშვნელობა ამ შემთხვევაში? როგორც ავღნიშნეთ ლექსიკური სეგმენტი ცვლადის სეგმენტს განსაზღვრავს იმ ადგილით სადაც ეს ცვლადი იყო გამოცხადებული, `var a=2`-ს გამოცხადებულია გლობალურ სეგმენტში, ამიტომ მისი ლექსიკური სეგმენტი იქნება გლობალური და console-ში დაიბეჭდება `2`

#### Lexical Scope vs Dynamic Scope

თუ სტატიკური სეგმენტი (Lexical Scope) ცვლადის მნიშვნელობას განსაზღვრავს სეგმენტიდან (Scope-იდან) სადაც ის შეიქმნა, დინამიური სეგმენტი (Dynamic Scope) ცვლადის მნიშვნელობას განსაზღვრავს ადგილიდან საიდანაც იყო ის გამოძახებული.

განვიხილოთ წინა მაგალითი და წარმოვიდგინოთ რომ JavaScript-ში ლექსიკურის მაგივრად დინამიური სეგმენტი გვაქვს, რა იქნება დაიბეჭდება console-ში?
```js
function foo() { 
  console.log( a ); 
}


function bar() { 
  var a= 3;
  foo(); 
}


var a=2; 


bar();
```

`a` ცვლადი `foo` ფუნქციაში არ გვაქვს, ლექსიკური სეგმენტის შემთხვევაში კომპილატორი ნახულობს რომელ სეგმენტშია შექმნილი ფუნქცია და იმ სეგმენტიდან იღებს მნიშვნელობას, ხოლო დინამიური სეგმენტის შემთხვევაში კომპილატორი Call Stack-ს [(Call Stack-ზე სტატია შეგიძლიათ წაიკითხოთ აქ](https://nikolozz.github.io/blog/JavaScript-V8-Call-Stack-Overview/) აყვება და ნახავს სად არის ფუნქცია გამოძახებული, რადგან `foo` გამოძახებულია `bar`-ში და მის სეგმენტში ცვლადის მნიშვნელობა არის `3` კონსოლში დაიბეჭდება `3`.

### Scope Chain
ყველა სეგმენტს აქვს ლინკი მის მშობელ სეგმენტზე, ამიტომ თუ JavaScript-ი ვერ იპოვის ცვლადს სეგმენტში სადაც კოდი სრულდება ის მოძებნის მშობელ სეგმენტში სანამ გლობალურ სეგმენტს არ მიაღწევს, თუ გლობალურ სეგმენტშიც არ მოიძებნება ცვლადი, მივიღებთ `ReferenceError`-ს.

![](https://miro.medium.com/max/1838/1*yGfmo21OoOTPAg0KAxU5GA.png)


### Closure 
Closure არის როცა შიდა ფუნქცია იმახსოვრებს გარე ფუნქციის ცვლადებს.

JavaScript-ში ფუნქცია ყოველი გამოძახებისას ახლიდან ქმნის მის ცვლადებს:

```js
function foo() {
    let counter = 0;
    counter += 1;
    console.log(counter);
}

foo() //1
foo() //1
```

ხოლო ამ შემთხვევაში:

```js
function parent() {
    let counter = 0;
    return function child() {
        counter += 1;
        console.log(counter);
    }
}

const fn = parent();
fn() // 1
fn() // 2
```

`child` ფუნქცია იმახსოვრებს მის მშობელ ფუნქციაში გამოცხადებულ ცვლადს, ამას ეწოდება `Closure`.

```js
function parent() {
    let counter = 0;
    return function child() {
        counter += 1;
        console.log(counter);
    }
}

const fn = parent();
const fn2 = parent();
fn() // 1
fn2() // 1
fn() // 2
fn2() // 2
```

როცა `fn2`-ის შემდეგ გამოვიძახეთ `fn` არ მივიღეთ `3` რადგან `fn`-ს და `fn2`-ს თავის Closure აქვთ, შესაბამისად `counter` ცვლადიც თავისი აქვთ Closure-ში. 



ფოტოს წყაროები:
- https://javascript.plainenglish.io/how-scope-chain-is-determined-in-javascript-b180eceae002
