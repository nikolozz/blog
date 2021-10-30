---
layout: post
title: NodeJS | Overview
published: true
tags: NestJS
---

NodeJS არის გარემო სადაც JavaScript-ი სრულდება და გვაძლევს საშუალებას დავწეროთ ბექენდი JavaScript-ზე, სხვა ვებ-სერვერებისგან განსხვავებით NodeJS-ი ერთ thread-იანია რადგან JavaScript-ში ერთი thread-ი გვაქვს, მაგრამ რატომ არის NodeJS ასეთი სწრაფი თუ thread-ი მხოლოდ ერთია? 

ამ სტატიაში დავწერ NodeJS-ის შემადგენელ კომპონენტებზე.

## NodeJS Overview
Node შედგება რამოდენიმე კომპონენტისგან
- Libuv
- V8
- Bindings
- c-areas, http, OpenSSL, zlib ...
![Image](https://raw.githubusercontent.com/khaosdoctor/my-notes/master/node/assets/nodejs-components.png)

განვიხილოთ ეს კომპონენტები ცალკცალკე, შემდეგ თვითოეულზე დეტალურად დავწერ.

- Your Code - ეგრედწოდებული User Land კოდი რომელსაც ჩვენ ვწერთ.
- Core Module - NodeJS-ის native მოდულები რომლებიც C++-ზეა რეალიზებული მაგალითად "fs" მოდული რომელიც ფაილებთან სამუშაოდ გვჭირდება.
- C++ Binding - JavaScript-ს არ აქვს წვდომა ოპერაციულ სისტემასთან ამიტომ ის C++ Binding-ების საშუალებით ახდენს კომუნიკაციას NodeJS-სთან fs.readFile რეალიზებულია Core Module-ში C++ -ზე, როცა ამ ფუნქციას ვიძახებთ Binding-ები ჩვენს JavaScript კოდს C++ -ის რეალიზაციასთან აკავშირებს.
- V8 - JavaScript Engine რომელიც ჩვენს დაწერილ JavaScript-ს ასრულებს, გამოყოფს მეხსიერებას (Stack, Heap) და ასევე ასუფთავებს მეხსიერებას ობიექტებისგან რომლებიც აღარ გამოიყენება (Garbage Collector). მისი 70% C++ -ზეა დაწერილი, დანარჩენი 30% JavaScript-ზე.
- Libuv - C-ზე დაწერილი ბიბლიოთეკაა, რომელიც გვაძლევს "არა ბლოკირებულ" I/O აბსტრაქციას (რადგან Node მუშაობს როგორც Windows-ზე, ასევე Linux-ზე და OSX-ზე, libuv გვაძლევს აბსტრაქციას რომ განურჩევლად ოპერაციული სისტემისა, შევძლოთ I/O ასინქრინული ოპერაციები შევასრულოთ რომლის რეალიზაცია OS-ებში შეიძლება განსხვავდებოდეს)

## User Land, Core Modules, Bindings

![](https://miro.medium.com/max/500/1*5vZoSm3Bx5VW_AoI6weOOQ.png)

როგორც უკვე ავღნიშნე JavaScript-ს არ აქვს წვდომა ოპერაციულ სისტემასთან რომ ფაილი წაიკითხოს/ჩაწეროს , NodeJS-ში გვაქვს მოდულები რომლებსაც აქვთ ოპერაციულ სისტემასთან წვდომა მაგალითად - os  - გვაძლევს ინფორმაციას ოპერაციული სისტემაზე, thread-ების რაოდენობაზე და ა.შ. path მოდული განსაზღვრავს ფაილების მდებარეობას, crypto მოდული რომლითაც შეგვიძლია დავაგენერიროთ ჰეში CPU-ს სიხშირით და ა.შ.

 ![](https://miro.medium.com/max/700/1*YMogMH_Bq2zjVemA7QsENQ.png)

[Node](https://github.com/nodejs/node)-ის რეპოზიტორიაში გვაქვს /lib და /src ფოლდერები /lib ფოლდერში აღწერილია API რომელსაც ჩვენ ვიყენებთ როდესაც ვიძახებთ fs.readFile()-ს. /src ფოლდერში კი readFile-ის იმპლემენტაციაა C++ -ზე.

Binding-ის საშუალებით ჩვენ ვაერთიანებთ JavaScript-ის და C++ -ის სამყაროს. fs.readFile გამოძახებისას internalBinding აკავშირებს C++ -ის readFile-ის რეალიზაციას (რომელიც /src ფოლდერშია განთავსებული) და libuv-ის მეშვეობით წვდომა გვაქვს ოპერაციულ სისტემაში არსებულ ფაილზე.

მაგალითისთვის ავიღოთ crypto.pbkdf2-ს გამოძახება რომელიც ჰეშს აგენერირებს.

![](https://o.quizlet.com/jM8hyzlspbNMup1gzxohew_b.png)

1. ვიძახებთ /lib ფოლდერში არსებულ pbkdf2 ფუნქციას
2. /lib ფოლდერში არსებული pbkdf2 internalBinding-ით იძახებს ამ ფუნქციის C++ რეალიზაციას
3. რადგან ჰეში პროცესორის სიხშირით გენერირდება, ფუნქციას ვარეგისტირებთ Thread Pool-ში სადაც ჰეშის გამოთვლა ხდება
4. Thread Pool-ი დააბრუნებს callback-ს Event Queue-ში რომელსაც Event Loop მოათავსებს Stack-ში ეს callback შესრულდება და მივიღებთ ჰეშის მნიშვნელობას

(Event Loop-ზე და Thread Pool-ზე შემდეგ სტატიაში დეტალურად დავწერ)

## V8

![](https://github.com/khaosdoctor/my-notes/raw/master/node/assets/v8-simplified.png)

V8 JavaScript Engine-ია რომელიც სხვა ინტერპრეტირებადი Engine-ებისგან განსხვავებით JavaScript აკომპილირებს Just in Time კომპილატორით. V8 გამოყოფს მეხსიერებას (stack-ს და heap-ს), ასევე ახდენს Garbage Collectings ანუ ობიექტების მეხსიერებიდან წაშლას რომლებსაც არ აქვთ ლინკი მშობელ ობიექტთან.

### Call Stack

როცა ვამბობთ რომ JavaScript-ში ერთი Thread გვაქვს ვგულისხმობთ რომ გვაქვს ერთი Call Stack სადაც ჩვენი კოდი სრულდება.

Stack არის Data Structure სადაც LIFO (Last In First Out) პრინციპით სრულდება ფუნქციები.

მაგალითად ავიღოთ შემდეგი კოდი:

```js
function multiply(n) {
    return n * n
}

function printSquare(num) {
    const s = multiply(num);
    console.log(s)
}

printSquare(5)
```
![](https://github.com/khaosdoctor/my-notes/raw/master/node/assets/simple-callstack.png)

1. შესრულდება printSquare(5)
2. შესრულდება multiply(5, 5)
3. ცვლად "s"-ს მიენიჭება მნიშვნელობა 5 (Stack-ში ინახება პრიმიტიული დატის ტიპები)
4. შესრულდება console.log(5)
5. დაბრუნდება undefined (რადგან არაფერს არ ვაბრუნებთ)

თითო ბლოკს Stack Frame ქვია. როგორც ავღნიშნე Stack-ში ინახება პრიმიტივები (string, number, undefined, symbol, null, boolean) ხოლო Heap-ში ინახება ობიექტები. Stack-ში ასევე ინახება reference ობიექტზე. 

Stack-ისთვის Node-ში გამოყოფილია 1MB მახსოვრობა, თუ ფუნქცია რეკურსიულად გამოვიძახეთ და 1MB გაცდა მივიღებთ შეცდომას.

```js
function foo() {
    foo()
}

foo()
```
```
VM198:2 Uncaught RangeError: Maximum call stack size exceeded
```
![](https://github.com/khaosdoctor/my-notes/raw/master/node/assets/stack-overflow.png)


ასე რომ როცა ვამბობთ რომ JavaScript ერთ thread-იანია, ვგულისხმობთ რომ მას მხოლოდ ერთი ოპერაციის შესრულება შეუძლია, დროის ერთეულში. 

## Libuv

ისეთი ოპერაციები როგორიცაა File Read/Write, networking სრულდება libuv-ში. libuv არის აბსტრაქცია ოპერაციულ სისტემაცე და გვაძლევს საშუალებას OS-თან ინტერაქციაში.

NodeJS-ის ერთ-ერთი მნიშვნელოვანი ნაწილია Event Loop-ი რომელიც libuv-ში არის რეალიზებული რომლის საშუალებით შეგვიძლია "არა ბლოკირებადი" I/O ოპერაციები შევასრულოთ, მიუხედავად იმისა რომ JavaScript მხოლოდ ერთ thread-იანია.

![](https://i.stack.imgur.com/QRePV.jpg)

Event Loop-ი CPU Intensive, I/O ოპერაციებისთვის არეგისტრირებს handler-ებს და გადასცემს Thread Pool-ში სადაც ეს ივენთები სრულდება, როცა ოპერაცია დასრულდება Event Loop დააბრუნებს მას Event Queue-ში საიდანაც შესრულებული callback-ი Stack-ში მოხვდება.

Event Loop და Libuv კომპლექსური თემებია, რომლებზეც შემდეგ სტატიებში დავწერ. 

ფოტოს წყაროები:
 - https://dev.to/khaosdoctor/node-js-under-the-hood-2-understanding-javascript-48cn
 - https://medium.com/swlh/node-js-c-da454904811f
 - https://stackoverflow.com/questions/36766696/which-is-correct-node-js-architecture