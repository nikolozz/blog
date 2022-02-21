---
layout: post
title: NodeJS | Event Loop
published: true
tags: NodeJS
---

NodeJS-ში ერთ-ერთი ყველაზე გაუგებარი ნაწილი Event Loop-ია ხშირად NodeJS Event Loop-ის განმარტების დროს საუბრობენ JavaScript-ის Event Loop-ზე, JavaScript-ის Event Loop რეალიზებულია ბრაუზერში და ის განსხვავდება NodeJS Event Loop-ისგან რომელიც რეალიზებულია Libuv-ში.

ჩვენ ვამბობთ რომ Node არის Event Driven, არაბლოკირებადი ჯავასკრიპტის შესრულების გარემო, სწორად Event Loop გვაძლევს საშუალებას რომ ასინქრონული ოპერაციები შევასრულოთ.

ამ სტატიაში დავწერ თუ რომელ "დიზაინ პატერნ"-ს იყენებს Node ასინქრონული ივენთების სისტემისთვის, რა არის Libuv და რა ფაზებისგან შედგება Event Loop-ი Node-ში.

### Reactor Pattern

![](https://i.stack.imgur.com/GtSae.jpg)

Node არის Event Driven პლატფორმა რომელიც დაფუძნებულია Reactor Pattern-ის დიზაინზე (Reactor Pattern არ არის კონკრეტული იმპლემენტაცია, ეს არის აბსტრაქცია Event Driven სისტემაზე).
ყველა ასინქრონული ოპერაცია იწვევს ივენთს, რომელიც Event Demultiplexer-ში რეგისტრირდება, როცა ოპერაციული სისტემა შეასრულებს ივენთს ის Event Queue-ში ხვდება საიდანაც დასრულებული ივენთის ჰენდლერი Stack-ში გადადის და კოდი სრულდება.

ზემოთ მოცემულ სურათში აღწერილია Reactor Pattern-ის მუშაობის პრინციპი

1. ვითხოვთ I/O ოპერაციის შესრულებას, ეს შეიძლება იყოს ფაილის წაკითხვა, crypto ოპერაცია და ა.შ. ამ ოპერაციას ვარეგისტრირებთ Event Demultiplexer-ში რომელიც მას მიანიჭებს "ჰენდლერს" რომელიც შესრულდება Stack-ში როცა ოპერაცია დასრულდება. 

2. დასრულებული I/O ოპერაცია გადადის Event Queue-ში სადაც FIFO პრინციპით არის განთავსებული ივენთები.

3. Event Loop იტერირებს ივენთებზე რომლებიც ივენთების რიგშია (Event Queue)

4. Event Loop-ი შეასრულებს რიგში არსებული ივენთების "ჰენდლერებს".

5. (5a) - როცა ჰენდლერები შესრულდება, ივენთ ლუპი გააგრძელებს მუშაობას (თუ ყველა რიგი ცარიელია Event Loop დაასრულებს მუშაობას და გამოვალთ პროგრამიდან). (5b) შესრულებულმა ჰენდლერებმა შეიძლება დაარეგისტრირონ ახალი ივენთები რომლებიც ასევე მოხვდებიან Event Demultiplexer-ში.

6. როცა Event Queue (ივენთების რიგი) ცარიელია, ვამოწმებთ Event Demultiplexer-ს თუ მასში დარეგისტირებულია ასინქრონული ოპერაციები იწყება შემდეგი Event Loop-ის იტერაცია, წინააღმდეგ შემთხვევაში გამოვდივართ პროგრამიდან.

Reactor Pattern არის დიზაინ პატერნი რომლის რეალიზაცია NodeJS-ში უფრო კომპლექსურია ვიდრე ზემოთ მოცემულ ალგორითმში. Event Demultiplexer არ არის გარკვეული კომპონენტი, ასევე ნოუდში არ გვაქვს მხოლოდ ერთი Event Queue და I/O არ არის ერთადერთი ივენთის ტიპი რომელიც Event Demultiplexer-ში ხვდება.

#### Event Demultiplexer
Event Demultiplexer-ი არის აბსტრაქტული კონცეპცია Reactor Pattern-ში, რომელიც რეალურად არ არსებობს, მისი კონკრეტული რეალიზაცია ხდება ოპერაციულ სისტემაში, ყველა ოპერაციული სისტემას ასინქრონული ოპერაციების შესასრულებლად თავისი მექანიზმი აქვს მაგალითად kqueue (macOs), epoll (Linux), IOCP (Windows) და ა.შ.

#### Event Queue 
NodeJS არ გვაქვს მხოლოდ ერთი ივენთების რიგი (Event Queue) setTimeouts, I/O, setImeddiate, Close Handlers, Promise, nextTick ივენთებს თავის რიგები აქვთ, რომელსაც ივენთ ლუპი გარკვეული თანმიმდევრობით ასრულებს (დეტალურად ამ სტატიაში განვიხილავ).


### Libuv
![](http://docs.libuv.org/en/v1.x/_images/architecture.png)

როგორც უკვე ვახსენე NodeJS ოპერაციების ასინქრონულად შესასრულებლად იყენებს ოპერაციული სისტემის მექანიზმს, ყველა ოპერაციულ სისტემაში მათი რეალიზაცია განსხვავდება, ამიტომ NodeJS-ისთვის შეიქმნა ბიბლიოთეკა Libuv რომელიც აბსტრაქციაა ოპერაციული სისტემის ასინქრონულ მექანიზმთან სამუშაოდ, Libuv-ის დამსახურებით როცა ჩვენ კოდს ვწერთ არ გვაინტერესებს რომელ ოპერაციულ სისტემაზე გაეშვება ჩვენი პროგრამა. 

ოპერაციულ სისტემებში network I/O სრულიად ასინქრონულია, ჩვენ შეგვიძია არ დაველოდოთ როდის იპოვის network-ი IP მისამართს, გააგზავნის request-ს სერვერზე და ა.შ. Linux-ში ფაილის ჩაწერა/წაკითხვა არ არის ასინქრონული ოპერაცია, მაგრამ Event Loop–ი არ იბლოკება, იმიტომ რომ Libuv-ში არის რეალიზებული Thread Pool-ი რომელიც დეფოლტად 4 thread-ს გვაძლევს ბლოკირებადი ოპერაციების შესასრულებლად როგორიცაა dns.lookup, file I/O, zlib, ასევე CPU ინტენსიური ოპერაციები რომლებსაც შეუძლია Event Loop-ი დაბლოკოს (როგორიცაა crypto.randomBites, crypto.pkdf2) სრულდება Thread Pool-ში.

**ყველა ასინქრონული ოპერაცია *არ* სრულდება Thread Pool-ში, NodeJS-ი Thread Pool-ში ასრულებს მხოლოდ იმ ოპერაციებს რომლის ასინქრონულად შესრულების საშუალებას ოპერაციული სისტემა არ გვაძლევს, სხვა დანარჩენი ასინქრონული ოპერაციები სრულდება ოპერაციული სისტემის ასინქრონულ მექანიზმში (epoll, kqueue, IOCP, event ports...)**

### Event Queues

როგორც უკვე ვახსენე NodeJS გვაქვს 6 ივენთის რიგი -
expired timeouts, I/O, setImeddiate, close event handlers, process.nextTick, microtasks მათ ყველას თავისი რიგი (Queue) გააჩნიათ, ეს რიგებია:

1. Expired Timers and Intervals queue - ეს არის setTimeout და setInterval-ის callback–ები, ისინი ოპერაციულ სისტემაში სრულდება, როცა ოპერაციული სისტემა შეამოწმებს დროს და ეს დრო მეტია ვიდრე setTimeout-ში გადაცემული დროის ერთეულიდან ათვლილი დრო ამ setTimeout/setInterval-ის callback მოხვდება Expired Timers-ის რიგში (queu-ში).

2. I/O Queue - დასრულებული I/O ქოლბექები. 

3. Imeddiate Queue - setImeddiate-ში გადაცემული callback-ების რიგი.

4. Close Handlers Queue - ყველა `close` ივენთის callback-ი. (მაგალითად connection.on('close', callback)

ამ რიგების გარდა გვაქვს ორი შუალედური რიგი, რომელიც არ არის libuv-ის ნაწილი, ისინი Node-შია იმპლემენტირებული. 

1. Next Tick Queue - process.nextTick-ში დარეგისტრირებული callback-ები.

2. Microtasks Queue - promise-ის შესრულებული callback-ები (resolved/rejected).

ისინი შუალედურია რადგან ყოველი ფაზის დამთავრების შემდეგ სრულდებიან მაგალითად როცა setTimeout/setInterval ივენთის ჰენდლერები შესრულდებიან და Expired Timeouts and Intervals რიგში მეტი ივენთი აღარ იქნება ეს ნიშნავს ფაზის დასასრულს, ფაზის დასრულების შემდეგ Next Tick Queue და Microtasks Queue რიგებში არსებული ივენთები შესრულდება და ასე ყოველი ფაზის დასრულების შემდეგ.

Node 11 ვერსიამდე, Micro Task/Next Tick callback-ები სრულდებოდნენ ფაზის დაწყებამდე, Node 11 > ვერსიებში process.nextTick/promise-ები შესრულდებიან აგრეთვე setTimeout/setInterval/setImmediate callback-ებს შორის.

```js
setTimeout(() => console.log('timeout1'));
setTimeout(() => {
    console.log('timeout2')
    Promise.resolve().then(() => console.log('promise resolve'))
});
setTimeout(() => console.log('timeout3'));
setTimeout(() => console.log('timeout4'));
```
Node version <= 11
```
timeout1
timeout2
timeout3
timeout4
promise resolve
```

Node version > 11
```
timeout1
timeout2
promise resolve
timeout3
timeout4
```

![](https://raw.githubusercontent.com/nikolozz/blog/master/images/Untitled.drawio.png)

Next Tick Queue-ს უფრო მაღალი პრიორიტეტი აქვს ვიდრე Micro Tasks Queue-ს, თუ პრომისები ან nextTick-ები რეკურსიულად დავამატეთ, ივენთ ლუპი შემდეგ ფაზაზე აღარ გადავა.

შემდეგ სტატიაში დეტალურად დავწერ ყველა Queue-ზე.

ფოტოს წყაროები:
- https://stackoverflow.com/questions/56622353/how-does-reactor-pattern-work-in-node-js
- http://docs.libuv.org/en/v1.x/_images/architecture.png
