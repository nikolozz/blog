---
layout: post
title: JavaScript | V8 and Call Stack Overview 
published: true
tags: JavaScript
---

JavaScript-ი ერთ-ერთი ყველაზე პოპულარული პროგრამირების ენაა რომელიც როგორც Frontend-ში ასევე Backend-ში გამოიყენება, ხშირად პროგრამისტები წერენ JavaScript-ზე, მაგრამ არ იციან როგორ მუშაობს JS შიგნიდან. ამ პოსტების სერიაში დავწერ JavaScript-ის მუშაობის პრინციპებზე.

თუ [წინა პოსტები](https://nikolozz.github.io/blog/NodeJS-Overview/) გაქვთ წაკითხული იცით რომ NodeJs-თვის JavaScript როგორც პროგრამირების ენა აირჩიეს რადგან ის ერთ ნაკადიანია. როცა ვიძახით რომ JavaScript ერთ ნაკადიანია, ეს ნიშნავს რომ Call Stack მხოლოდ ერთი გვაქვს, რადგან უკეთესად გავერკვეთ, გავარჩიოთ თუ როგორ სრულდება JavaScript კოდი.

(V8 და Call Stack-ი [NodeJs პოსტების სერიაშიც](https://nikolozz.github.io/blog/NodeJS-Overview/) არის განხილული, მაგრამ აქცენტი NodeJs-ზეა გაკეთებული, JS სერიაში თვითონ ენის მუშაობის პრინციპებზე დავწერ)

JavaScript-ი სრულდება გარკვეულ გარემოში - ბრაუზერში, NodeJs-ში, IOT (დღესდღეობით JS ისეთი პოპულარულია რომ ტოსტერებშიც გვხვდება) ვიცით რომ JS მაღალი დონის ენაა, შესაბამისად ჩვენი დაწერილი კოდი უნდა გარდაიქმნას ბაიტ კოდათ, კომპიუტერისთვის გასაგებ ენაზე, ამ გარდაქმნას ასრულებს JS Engine რომლებიცაა: 
- V8 - გვხვდება Chrome-ში და NodeJS-ში.
- SpiderMonkey - დაწერილია Mozilla Firefox-ისთვის.
და ა.შ. ამ პოსტში განვიხილავთ V8-ს.

### JS Engine
V8 შექმნა Google-მა და როგორც ავღნიშნეთ ის Chrome-ის გარდა ასევე NodeJs-ში გხვდება. ძველი JS Engine-ბი ინტერპეტირებადი იყო, რაც ნიშნავს რომ line by line კითხულობდნენ კოდს რაც პროგრამის ნელ შესრულებას განაპირობებდა. V8 არის JIT კომპილატორი (Just In Time) ამაზე უფრო ვრცლად ვისაუბრებთ, ამ ეტაპზე შეგვიძლია გამარტივებულად ასე წარმოვიდგინოთ:

![](https://github.com/khaosdoctor/my-notes/raw/master/node/assets/v8-simplified.png)

- Heap - მეხსიერება სადაც ინახება ობიექტები.
- Call Stack - პროგრამის შესრულების დროს გამოძახებული ფუნქციები/ოპერაციები ხვდება Call Stack-ში და თითო ასეთ ბლოკს ქვია Stack Frame - რომელშიც ლოკალური ცვლადები ინახება.

### Call Stack
როგორც ავღნიშნეთ JS ერთ ნაკადიანია და ამიტომ ერთი Call Stack გვაქვს, ეს იმას ნიშნავს რომ დროის გარკვეულ ერთეულში მხოლოდ ერთი ოპერაციის შესრულება შეგვიძლია.

Call Stack - მონაცემთა სტრუქტურაა, რომელშიც პირველი შესული ფუნქცია გადის ბოლო (LIFO - Last In First Out) შეგვიძლია წარმოვიდგინოთ რომ მას აქვს ორი მეთოდი push და pop, push ახალ ფუნქციას დაამატებს თავში, ხოლო pop წაშლის.

![](https://i0.wp.com/learnersbucket.com/wp-content/uploads/2018/12/stack-2-1.png?fit=768%2C400&ssl=1)

ვნახოთ Call Stack შემდეგი კოდისთვის:

```js
function multiply(x, y) {
    return x * y;
}
function printSquare(x) {
    var s = multiply(x, x);
    console.log(s);
}
printSquare(5);
```

![](https://habrastorage.org/r/w1560/getpro/habr/post_images/19c/a4b/bad/19ca4bbadd85f5c38bcfa0a87a79bc75.png)

1. შესრულდება printSquare არგუმენტით "5" და Stack Frame-ში გვექნება ეს ფუნქცია სადაც ასევე მისი არგუმენტი შეინახება მეხსიერებაში.
2. printSquare გამოიძახებს multiply ფუნქციას, რომლის შესრულების რეზულტატი ჩაიწერება ცვალდში "s" (მისთვის მეხსიერება გამოყოფილია printSquare-ის Stack Frame-ში).
3. Stack-იდან "ამოვარდება" multiple რადგან ის უკვე შესრულდა და შესრულდება console.log-ი.
4. console.log-ი შესრულების შემდეგ "ამოვარდება" Stack-იდან და დავბრუნდებით printSquare ფუნქციაზე.
5. რადგან console.log ბოლო ოპერაცია იყოს printSquare ფუნქციაში, printSquare-მა დაასრულა შესრულება და ამოვარდება Stack-იდან.

როცა ფუნქცია იშლება Stack-იდან მისი ცვლადებისთვის გამოყოფილი მეხსიერებაც თავისუფლდება.

Heap-ისგან განსხვავებით Stack-ს აქვს მეხსიერების ლიმიტი, Heap პროგრამის შესრულების დროს შეიძლება დინამიურად გაიზარდოს ხოლო Stack-ისთვის გამოყოფილი მეხსიერება უცვლელია, თუ ამ მეხსიერებას გადავაჭარბებთ მოხდება Stack Overflow.

```js
function foo() {
    foo();
}
foo();
```

მაგალითად ზემოთ მოცემულ კოდში რეკურსიულად ვიძახებთ foo()-ს.
![](https://habrastorage.org/r/w1560/getpro/habr/post_images/24d/31f/f43/24d31ff435926a3e94a4b1a169a69d43.png)

ამ კოდის შედეგი იქნება:
``` 
Uncaught RangeError: Maximum call stack size exceeded.
```

ასევე როცა პროგრამა ისვრის Exception-ს შეგვიძლია მარტივად მივაგნოთ სად მოხდა შეცდომა, რადგან ვიცით რომელმა ფუნქციას "გაისროლა" Exception-ი და სად იყო ეს ფუნქცია გამოძახებული, ეს მონაცემები გვაძლევს Stack Trace-ის.

```js
function foo() {
    throw new Error('Program Crashed');
}
function bar() {
    foo();
}
function start() {
    bar();
}
start();
```

```
VM488:2 Uncaught Error: Program Crashed
    at foo (<anonymous>:2:11)
    at bar (<anonymous>:5:5)
    at start (<anonymous>:8:5)
    at <anonymous>:10:1
```

Stack Trace-ის მეშვეობით ვხედავთ რომ შეცდომა მოხდა foo-ში, რომელიც გამოიძახა bar-მა, რომელიც გამოიძახა start-მა :) 

ეს იყო V8-ს მცირედი მიმოხილვა, შემდეგ სერიებში დეტალურად დავწერ V8-ზე.
