---
layout: post
title: NodeJS | Event Loop - nextTick, Promises, Timers, IO
published: true
tags: NodeJs
---

[წინა პოსტში](https://nikolozz.github.io/blog/NodeJS-Event-Loop-Phases/) განვიხილეთ Event Loop-ის ფაზები და მათი შესრულების თანმიმდევრობა ამ პოსტში ცალ-ცალკე განვიხილავ ყველა ფაზას.

![](https://raw.githubusercontent.com/nikolozz/blog/master/images/Untitled.drawio.png)

დავუბრუნდეთ ამ სურათს, აქ ნაჩვენებია ყველა რიგი (Queue) მათი თანმიმდევრობის მიხედვით, process.nextTick და Promise რიგები NodeJS-შია რეალიზებული ხოლო სხვა დანარჩენი Libuv-ის ნაწილია (ამაზე ვრცლად [ამ სტატიაში](https://nikolozz.github.io/blog/NodeJS-Event-Loop/) წერია). 
როგორც წინა სტატიებში ავღნიშნეთ Libuv-ში ხვდება ის Event-ები რომლებსაც ოპერაციული სისტემის ასინქრონული მექანიზმი ასრულებს, nextTick და Micro Task-ი კი NodeJS-ში სრულდება. 

#### Intermediate Queues - process.nextTick, Micro Tasks.
როცა Event Loop-ი იწყებს იტერაციას ის პირველ რიგში შეამოწმებს Intermediate Queue-ს (process.nextTick, Micro Tasks) ხოლო შემდეგ გადავა Libuv-ში რეალიზებულ რიგებზე, ყოველი ფაზის დასრულების შემდეგ Event Loop-ი შეამოწმებს process.nextTick და Micro Task რიგებს.

```js
console.log('Start')
process.nextTick(() => console.log('process.nextTick'));
setTimeout(() => {
  console.log('setTimeout')
  Promise.resolve().then(() => console.log('Promise'))
}, 0);
setImmediate(() => console.log('setImmediate!'))
```

ამ კოდის შესრულების შედეგი იქნება (node version > 11)

```
Start
process.nextTick
setTimeout
Promise
setImmediate!
```

პირველ რიგში დაილოგება `Start` რადგან ის Event Loop-ის იტერაციის დაწყებამდე მოხვდება Stack-ში და შესრულდება. შემდეგ დარეგისტრირდება Event-ები nextTick, setTimeout, setImmediate. როგორც ვთქვით პირველი სრულდება Intermediate Queue-ები, ამიტომ Stack-ში პირველი nextTick-ის callback-ი მოხვდება და დაილოგება process.nextTick, Event Loop-ი შეამოწმებს Micro Tasks რიგს, რადგან Promise-ები არ გვაქვს ამ შემთხვევაში გადავა Expired Timers-ს რომელიც დალოგავს setTimeout და დაარეგისტრირებს ახალ Event-ს Micro Task რიგისთვის. **Event Loop-ი სანამ Immediate ფაზაზე გადავა შეამოწმებს Intermediate რიგებს**, setTimeout-ში დამატებული Promise-ის callback-ი შესრულდება და დალოგავს Promise, ხოლო შემდეგ უკვე გადავალთ Immediate ფაზაზე.
**იტერაციის დაწყებამდე და ყოველი ფაზის დასრულების შემდეგ Event Loop ასრულებს process.nextTick და Micro Task რიგებში არსებულ callback-ებს.**

### Expired Timers
Libuv-ში რეალიზებულია timers heap რომელშიც ვარდება setTimeout, setInterval-ის callback-ები. Expired Timers ფაზაზე Node შეამოწმებს timer heap-ში თუ არის expired timer-ი, თუ ამ დროისთვის გასულია timer-ში მითითებული დრო ამ ტაიმერის callback-ი შესრულდება. 
```js
setTimeout(cb, 1000) 
```
არ გვაძლევს გარანტიას რომ ეს timeout ზუსტად 1000 მილიწამში შესრულდება თუ Expired Timers ფაზამდე, რაიმე CPU ინტენსიური ტასკი სრულდებოდა და Event Loop დაბლოკილი იყო, შესაბამისად იქნება delay სანამ Event Loop გადავა Expired Timers ფაზაზე და შეასრულებს მასში არსებულ callbacks. 

### I/O
I/O ტასკებს ვეძახით იმ ტასკებს რომლებიც გარე მოწყობილეობებთან ურთიერთქმედებენ მაგალითად network დაფა ან SSD. JavaScript-ი მაღალი დონის პროგრამირების ენაა, რაც იმას ნიშნავს რომ მას არ აქვს წვდომა ფაილების სისტემასთან, ფაილებთან ოპერაციები რეალიზებულია Libuv-ში. Libuv-ი Thread Pool-ში გამოყოფს thread-ს ფაილთან სამუშაოდ, რადგან ფაილის წაკითხვა/ჩაწერა ოპერაციულ სისტემებში სინქრონულია და თუ thread-ში არ გავიტანთ Event Loop-ი დაიბლოკება.

```
fs.readFile(filename, cb)

fs.readFileSync(filename, cb)
```

fs.readFile-ის შემთხვევაში იქმნება thread-ი რომელიც კითხულობს ფაილს გარე მოწყობილობიდან. ხოლო fs.readFileSync-ისთვის არ იქმნება ცალკე thread-ი ამიტომ Event Loop-ი დაიბლოკება სანამ ფაილი არ იქნება წაკითხული.

### Immediates Queue
```js
setImmediate(() => {
   console.log('setImmediate');
});
```

setTimeout-ისგან განსხვავებით setImmediate გვაძლევს გარანტიას რომ ის შესრულდება I/O ფაზის დასრულების შემდეგ. 

მაგალითად:
```js
const fs = require('fs');

fs.readFile('text.txt', { encoding: 'utf-8' }, (err, cb) => {
    setTimeout(() => {
        console.log('setTimeout', cb)
    }, 0);
    setImmediate(() => {
        console.log('setImmediate', cb)
    })
});
```
რადგან ვიცით რომ ფაილის წაკითხვის შემდეგ I/O ფაზა დასრულდა, იმისდა მიუხედავად რომ setTimeout-ი 0 მილიწამია და ის მაშინვე უნდა შესრულდეს setImmediate შესრულდება, რადგან ის I/O ფაზის შემდეგ მოდის.

წინა პოსტებში ვრცლად ვისაუბრეთ Node-ზე, კითხვების ან რედაქტირების სურვილისთვის შეგიძლიათ [მომწეროთ](https://www.linkedin.com/in/nikoloz-bokeria/) :) 

- [NodeJS | Overview](https://nikolozz.github.io/blog/NodeJS-Overview/)
- [NodeJS | Event Loop](https://nikolozz.github.io/blog/NodeJS-Event-Loop/)
- [NodeJS | Event Loop Phases](https://nikolozz.github.io/blog/NodeJS-Event-Loop-Phases/)


