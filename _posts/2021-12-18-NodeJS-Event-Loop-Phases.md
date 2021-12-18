---
layout: post
title: NodeJS | Event Loop Phases
published: true
tags: NodeJs
---

JavaScript-ის Event Loop-ში გვაქვს 2 Queue ესენი არის Callback (Task) Queue და MicroTask Queue (Promises), NodeJS-ში გვაქვს 6 Queue რომლის Event Callback–ები გარკვეულ ფაზაზე სრულდება, ყველა ფაზას თავის Queue აქვს. [წინა პოსტში](https://nikolozz.github.io/blog/NodeJS-Event-Loop/) განვიხილე Event Loop-ის და Libuv-ის მუშაობის პრინციპი, ამ პოსტში დავწერ ფაზების თანმიმდევრობაზე.

### Event Loop Phases
როგორც ზემოთ ვახსენე NodeJS-ში გვაქვს 6 ფაზა, თითოეულს თავისი Event Queue აქვს სადაც დასრულებული callback-ები ვარდება. ექვსი ფაზიდან ოთხი ფაზა არის რეალიზებული Libuv-ში, ორი ფაზა კი Node-ში. 

Libuv Phases
- Expired Timer Callbacks
- I/O
- Set Immediate
- Close Handlers

NodeJS Phases
- Next Tick
- Promises

როდესაც Node-ის პროგრამა სრულდება, ინტერპრეტატორი კითხულობს კოდს და ასრულებს მას თუ კოდი ასინქრონულია იძახებს შესაბამის V8 Bindings-ს და Libuv-ი არეგისტრირებს Event-ს რომლის Callback-იც გარკვეულ ფაზაზე შესრულდება.

მაგალითად:
```js
console.log('start');
setTimeout(() => console.log('setTimeout'), 0);
console.log('end');
```
ამ კოდის რეზულტატი იქნება:
```
start
end
setTimeout
```
გავარჩიოთ step-by-step

1. პირველ ხაზზე პროგრამამ წაიკითხა console.log('start') რადგან ეს კოდი სინქრონულია თავსდება Stack-ში, სრულდება და კონსოლში იწერება 'start'.
2. ვიძახებთ setTimeout-ს (setTimeout-ი ასინქრონული ოპერაციაა).
3. V8 Binding-ის მეშვეობით ვიძახებთ მის რეალიზაციას Libuv-ში.
4. Libuv არეგისტრირებს setTimeout-ის callback-ს.
5. Stack-ში ვარდება console.log('end'), სრულდება და კონსოლში იწერება 'end'.
6. Libuv-ი ფონურ რეჟიმში ასრულებს setTimeout-ს (მის დეტალურ შესრულებაზე შემდეგ პოსტში დეტალურად დავწერ).
7. Timeout დროის გასვლის შემდეგ Libuv-ი მოათავსებს ამ ანონიმურ ფუნქციას `() => console.log('setTimeout')` Set Timeout-ის Event Queue-ში.
8. NodeJS-ში პირველი ფაზა Set Timeout არის, ამიტომ პირველი Set Timeout-ის Event Queue-ში მოთავსებული callack-ები შესრულდება, შესაბამისად Event Loop-ი ამ ანონიმურ ფუნქციას მოათავსებს Stack-ში.
9. Stack-ში შესრულდება console.log('setTimeout') რომელიც კონსოლში დაწერს 'setTimeout'-ს.

(შეგიძლიათ ეს კოდი [ამ საიტზე](https://www.jsv9000.app/) შეასრულოთ, რომელიც Event Loop-ის ვიზუალიზაციას ახდენს JavaScript-ისთვის, მაგრამ JavaScript-ის და Node-ის Event Loop-ები განსხვავდება)

ასინქრონული კოდი სხვადასხვა ფაზებზე სრულდება. განვიხილოთ მათი თანმდიდევრობა:

![](https://raw.githubusercontent.com/nikolozz/blog/master/images/Untitled.drawio.png)

NodeJS-ის ფაზებს Intermediate ფაზებს ვუწოდებთ რადგან მათში არსებული callback–ები ყოველი ფაზის დასრულების შემდეგ სრულდებიან. შესაბამისად ფაზების თანმიმდევრობა, როგორც სურათზეა ნაჩვენები, შემდეგნაირია:

1. Expired Timer Callbacks (setTimeout/setInterval)
2. I/O
3. Set Immediate
4. Close Handlers

Event Loop-ის მუშაობის დაწყებისას და ყოველი ფაზის დასრულების შემდეგ:
1. process.nextTick Callbacks
2. Micro Tasks

(ამ სტატიაში დაწერილი კოდი Node 14 ვერსიის გარემოში სრულდება, შეიძლება სხვადასხვა ვერსიაში მათი თანმიმდევრობა განსხვავდებოდეს, ამაზეც შემდეგ პოსტში დავწერ)

შევასრულოთ შემდეგი კოდი რომელშიც ყველა ფაზაზე მოხვდება callback-ი:

```js
const fs = require('fs')

setTimeout(() => console.log('3. Expired Timer Callbacks'), 1)

fs.readFile('./some.txt', { encoding: 'utf-8' }, () =>
  console.log('4. I/O'),
)

setImmediate(() => console.log('5. Set Immediate'))

process.nextTick(() => console.log('1. process.nextTick'))

Promise.resolve().then(() => console.log('2. Micro Task | Promise'))

process.on('exit', () => {
  console.log('6. Close Handler')
})
```
[(code snippet)](https://gist.github.com/nikolozz/35c568ec4cb5f082c6e58586f34bcae4)
ამ კოდის რეზულტატი იქნება:
```
1. process.nextTick
2. Micro Task | Promise
3. Expired Timer Callbacks
4. I/O
5. Set Immediate
6. Close Handler
```

გავარჩიოთ შემდეგი კოდი:
1. Node-მა შეასრულა კოდი line-by-line
2. დაარეგისრიდა Event-ები (ფაზა + callback) 
3. Libuv-იმ დაიწყო ასინქრონული event-ების შესრულება
4. Event-ების შესრულების შემდეგ Libuv-ი ანთავსებს Callback-ებს Queue-ში
5. Event Loop-ი იწყებ მუშაობას
6. პირველი სრულდება process.nextTick და promise callback-ები.
7. შემდეგ Expired Callbacks, I/O, Set Immediate, Close Handler callback-ები თანმიმდევრობით.

შეგვიძლია Queue-ები ასე წარმოვიდგინოთ:

![](https://raw.githubusercontent.com/nikolozz/blog/master/images/Event%20Loop%20Phases.drawio.png)

ჩნდება კითხვა თუ რა მოხდება setTimeout 5 წამამდე რომ გავზარდოთ? დაელოდება timeout-ის დასრულებას 5 წამი? გავზარდოთ setTimeout-ის დრო 5 წამამდე.

ავიღოთ შემდეგი კოდი:
```js
const fs = require('fs')
const net = require('net')
const server = net.createServer();
server.listen(8080);

setTimeout(() => console.log('7. Expired Timer Callbacks'), 5000)

fs.readFile('./some.txt', { encoding: 'utf-8' }, () => console.log('4. I/O'))

setImmediate(() => console.log('5. Set Immediate'))

process.nextTick(() => console.log('1. process.nextTick'))

Promise.resolve().then(() => console.log('2. Micro Task | Promise'))

const socket = net.createConnection(8080, '127.0.0.1');
socket.destroy();
socket.on('close', () => console.log('6. Socket Close Handler'));
```
[(code snippet)](https://gist.github.com/nikolozz/7b91d24347a73881adb57a42a693bf9c)
კოდის რეზულტატი იქნება შემდეგი:
```
1. process.nextTick
2. Micro Task | Promise
4. I/O
5. Set Immediate
6. Socket Close Handler
7. Expired Timer Callbacks
```

რადგან წინა კოდში setTimeout-ს 1 მილიწამიანი ტაიმაუტით ვიძახებდით, Event Loop-ის მუშაობის დაწყებისას მისი callback უკვე Expired Callback Queue-ში იყო, ამ შემთხვევაში მას 5 წამიანი ტაიმაუტით ვიძახებთ, შესაბამისად Event Loop-ი სხვა ფაზებზე გადადის, ბოლო ფაზა Close Handler-ის callback-ი სრულდება `Socket Close Handler`, Close Handlers ფაზის დასრულების შემდეგ Node ამოწმებს ხომ არ არის Pending Task-ი, ჩვენს შემთხვევაში setTimeout Pending Task არის ამიტომ Node პროგრამიდან არ გამოდის, setTimeout-ის დასრულების შემდეგ Event Loop-ი იწყებს ახალ იტერაციას Expired Callback-ის Event Queue-ში იქნება callback-ი რომელიც console-ში ჩაწერს `Expired Timer Callbacks`. კოდში გვიწერია `server.listen(8080);` Event Loop-ი გადავა მოლოდინის რეჟიმში (რადგან ივენთები არ არის), მაგრამ პროგრამა არ შეწყვეტს მუშაობას.

განვიხილოთ კიდევ ერთი კოდის მაგალითი:
```js
console.log('simple console.log')
process.nextTick(() => console.log('processNextTick'));
setImmediate(() => {
  console.log('setImmediate');
  setTimeout(() => console.log('setTimeout'), 0);
  Promise.resolve().then(() => console.log('second promise resolve'))
});
Promise.resolve().then(() => console.log('promise resolve'));
new Promise((resolve) => {
  console.log('new Promise()')
  resolve();
});
setTimeout(() => console.log('outer setTimeout'), 0);
```
[(code snippet)](https://gist.github.com/nikolozz/0620e8ce6370f3ee8796ee3bb123cba5)
ეცადეთ გასცეთ პასუხი რა თანმიმდევრობით შესრულდება კოდი და შეადარეთ რეზულტატს:

```
simple console.log
new Promise()
processNextTick
promise resolve
outer setTimeout
setImmediate
second promise resolve
setTimeout
```

1. Node იწყებს კოდის შესრულებას
2. შესრულდება სინქრონული simple console.log
3. შესრულდება process.nextTick და მისი ფაზისთვის დარეგისტრირდება Event-ი შემდები callback-ით: () => console.log('processNextTick').
4. შესრულდება setImmediate, დარეგისტრირდება Event-ი
5. შესრულდება Promise.resolve, დარეგისტრირდება Event-ი
6. შესრულდება new Promise(...), რადგან console.log-ს resolve()-ში არ გადავცემთ callback-ად, ის სინქრონული კოდის ნაწილია (იმის მიუხედავად რომ Promise-ის callback-ში წერია) ამიტომ console-ში ჩაიწერება 'new Promise()'
7. შესრულდება setTimeout, დარეგისტრირდება Event-ი
8. **process.nextTick, Micro Task** callback-ები შესრულდება -> 
    processNextTick
    promise resolve
9. **Expired Timeout Callback** - callback-ებში არის მე-7. ნაბიჯში დარეგისტრირებული Event-ის callback-ი, რომელიც დალოგავს 'outer setTimeout'
10. მოწმდება **I/O** Event-ებს, რომელიც ცარიელი იქნება, გადავდივართ შემდეგ ფაზაზე.
11. **Set Immediate** - შესრულდება setImmediate-ის callback-ი რომელიც 
    11.1 დალოგავს 'setImmediate'-ს
    11.2 სრულდება setTimeout-ი, რეგისტრირდება callback-ი Expired Callback Queue-ში
    11.3 სრულდება Promise-ი, რეგისტრირდება callback-ი Micro Task Queue-ში
9. setImmediate ფაზის (როგორც ყოველი ფაზის დასრულების შემდეგ) ვამოწმებთ **process.nextTick, Micro Task** Queue-ებს 
10. 8.3 ნაბიჯში დარეგისტრირებული Promise არის Micro Task Queue-ში, სრულდება მისი callback-ი, ილოგება 'second promise resolve'
11. **Close Handler** - ფაზაში ივენთი არ არის. რადგან Close Handlers ბოლო ფაზაა, Event Loop იწყებს ახალ იტერაციას.
12. მოწმდება **process.nextTick, Micro Task** Queue-ები, ივენთები არ არის.
12. **Expired Callbacks** - 8.2 ნაბიჯში დარეგისტრირებული setTimeout-ის callback-ი არის Expired Callback Queue-ში, სრულდება და ილოგება 'setTimeout'
13. შემდეგ არც ერთ ფაზაში არ იქნება ივენთები, ამიტომ Node დაასრულებს მუშაობას.

#### Event Loop Starvation 
როგორც უკვე ვახსენე process.nextTick და Micro Task-ები სრულდება ყოველი ფაზის დასრულების შემდეგ, რაც იმას ნიშნავს რომ სანამ ეს Queue-ბი არ დაცარიელდება, სხვა ფაზაზე არ გადავა Node.

```js
function promiseRecursive(count) {
  if (count === 0) return

  new Promise(() => {
    console.log('promise starvation')
    promiseRecursive(count - 1)
  })
}

setTimeout(() => console.log('setTimeout'), 0);
fs.readFile('./firebase.json', {encoding: 'utf-8'}, () => {
  console.log('readFile')
  promiseRecursive(5);
  setTimeout(() => console.log('setTimeout'), 0);
})
```

`promiseRecursive` ფუნქცია 5-ჯერ რეკურსიულად დაამატებს Promise-ს Micro Task Queue-ში

![](https://raw.githubusercontent.com/nikolozz/blog/master/images/Event%20Loop%20Starvation.drawio.png)

თუ ფუნქციას არ შევზღუდავდით 5 გამოძახებით, setTimeout Queue-ზე არასდროს გადავიდოდა Event Loop-ი.

ამიტომ კოდის შესრულების რეზულტატი შემდეგი იქნება:
```
setTimeout
readFile
promise starvation
promise starvation
promise starvation
promise starvation
promise starvation
setTimeout
```

ანალოგიურად process.nextTick ფაზაზე.

```js
function setImmediateRecursive(count) {
  if (count === 0) return

  setImmediate(() => {
    console.log('setImmediate')
    setImmediateRecursive(count - 1)
  })
}

setTimeout(() => console.log('setTimeout'), 0);
fs.readFile('./firebase.json', {encoding: 'utf-8'}, () => {
  console.log('readFile')
  setImmediateRecursive(5);
  setTimeout(() => console.log('setTimeout'), 0);
})
```

თუ იგივეს ვცდით setImmediate-ის შემთხვევაში, პირველი setImmediate-ის შესრულების შემდეგ Event Loop-ი გადავა შემდეგ ფაზაზე.

```
setTimeout
readFile
setImmediate
setTimeout
setImmediate
setImmediate
setImmediate
setImmediate
```

შემდეგ სტატიაში ვეცდები თვითოეული ფაზა ცალკ-ცალკე განვიხილო. 
