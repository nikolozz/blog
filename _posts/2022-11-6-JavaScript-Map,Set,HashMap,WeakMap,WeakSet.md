---
layout: post
title: JavaScript | Hashmap, Map, Set, Weak*
published: true
tags: JavaScript
---

ES6-ში დაემატა ახალი მონაცემთა სტრუქტურები Map და Set რომლებმაც მოგვცა ამოცანების გადაჭრის გზები Object და Array-ის გამოყენების გარეშე. Map არის მონაცემთა სტრუქტურა Key, Value მონაცემების შესანახად, Set კი მონაცემთა სტრუქტურაა სადაც ერთი და იგივე ტიპის, უნიკალური, მონაცემები ინახება თანმიმდევრულად. ამ სტატიაში განვიხილავთ Hashmap-ს, Map და Set მონაცემთა სტრუქტურებს, მოვიყვან პრაქტიკულ მაგალითებს და ასევე დავწერ WeakMap და WeakSet-ზე, რომლებსაც პრაქტიკაში ნაკლებად ვხვდებით, მაგრამ მათი მუშაობის პრინციპი უნდა ვიცოდეთ. 

## Hashtable and Hashmap
JavaScript-ში Map-ის დამატებამდე ვხმარობდით Object-ს როგორც Hashmap-ს, რომელიც ზუსტად არ არის Hashmap-ის იმპლემენტაცია, მას გააჩნია პროტოტიპში მეთოდები რომლებიც Hashmap მონაცემთა სტრუქტურაში ზედმეტია, ამიტომ JS-ს დაემატა Map-ი რომელიც HashMap-ის უფრო ზუსტი იმპლემენტაციაა.

Hashtable და Hashmap არის მასივის მაგვარი მონაცემთა სტრუქტურები სადაც ინდექსად "დაჰეშილი" Key ინახება. მათ შორის მთავარი განსხვავებაა რომ Hashtable-ის შეუძლია სინქრონულად განაახლოს Key-ები და ყველა ნაკადისთვის იყოს გაზიარებული. Hashmap-ი კი ერთი ნაკადისთვის არის ლიმიტირებული, ის JS-ში Map-ის სახით არის წარმოდგენილი.

Hashmap-ი როგორც Array ისე არის ორგანიზებული, შემავალი Key გადის ჰეშ ფუნქციას, რომელიც დააბრუნებს ინდექსს სადაც ეს Key შეინახება, ამიტომ მასში მოძებნა, ჩაწერა და წაშლა O(1) დროში მოხდება.

![](https://www.freecodecamp.org/news/content/images/2021/05/g983.jpg)
## Map

### What is Map

როგორც ზემოთ ავღნიშნე Map არის მონაცემთა სტრუქტურა, რომელიც JavaScript-ის გარდა ბევრ სხვა პროგრამირების ენაში გვხვდება, Map-ში ინახება Key, Value მონაცემები, სადაც Key უნიკალური არის მონაცემთა ტიპის მიხედვით. Map-დან ყველა ელემენტს O(1) დროში ვიღებთ.

### Map vs Object

მთავარი განსხვავება Object და Map-ს შორის არის ის რომ ობიექტში მხოლოდ `String` და `Symbol` მონაცემთა ტიპის Key-ები შეგვიძია შევინახოთ, ხოლო Map-ში ნებისმიერი მონაცემთა ტიპის შენახვა შეგვიძლია Key–ით.

Object

```javascript
const obj = {
    1: 'Number 1',
    '1': 'String 1'
}

console.log(obj[1]) // String 1
console.log(obj['1']) // String 1
```

Map

```javascript
const map = new Map([[1: 'Number 1'], ['1', 'String 1']])

console.log(map[1]) // Number 1
console.log(map['1']) // String 1

// Set Boolean data type as Key
map.set(true, 'Boolean');

console.log(map.get(true)) // Boolean
```

გარდა ამისა Map-ი Object-გან განსხვავებით იტერირებადია, შეგვიძლია `forEach`, `for of` მეთოდებით Map-ზე იტერირება.

```javascript
const object = {
    'x': 'String 1',
    'y': 'String 2'
}

for(const key of object) {
    console.log(key);
}
```

ამ შემთხვევაში რადგან ობიექტზე არ შეგვიძლია იტერირება JavaScript-ის შეცდომა გვექნება

```javascript
Uncaught TypeError: object is not iterable
```

ხოლო Map-ის შემთხვევაში შეგვიძლია იტერირება

`Map.prototype.forEach` - აბრუნებს [key, value] ტიპს callback პარამეტრად.

`Map.prototype.values` - დააბრუნებს Value-ებს Map-იდან

`Map.prototype.keys` - დააბრუნებს Key-ებს Map-იდან

ვრცლად Map-ის მეთოდებს შეგიძლიათ გაეცნოთ აქ - https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map

დავწეროთ რამოდენიმე მაგალითი:

```javascript
const map = new Map([
    ['x', 'String 1'],
    ['y', 'String 2']
])

// Iterate with for of
for(const key of map.values()) {
    console.log(key);
}

// Iterate with forEach
map.forEach((value, key) => {
  console.log(key);
})
```

ასევე Map-ი უფრო კარგი გადაწყვეტილებაა Key,Value-ს შესანახად რადგან Object-ს prototype-ში აქვს მეთოდები რომლებიც საერთოდ არ გვჭირდება ამ კონკრეტულ მონაცემთა სტრუქტურაში. ამის გარდა მარტივად შეგვიძლია ვნახოთ თუ მაპში გვაქვს ელემენტი `map.has(elem)`-ის გამოყენებით და გავიგოთ Map-ის სიგრძე `map.size` property-ით.

### Map practical case word counting

გასაუბრებებზე ხშირად გვხვდება ასეთი ამოცანა: 

მოცემული გვაქვს სიტყვა `hello` უნდა დავთვალოთ რამდენჯერ მეორდება თვითოუელი ასო ამ სიტყვაში.

ყველაზე მარტივი გზა რაც პირველი გვაფიქრდება არის O(n^2) ციკლი, სადაც ერთი ციკლი უშუალოდ სიტყვის ბგერებზე მოახდენს იტერაციას ხოლო მეორე ციკლი დაითვლის რამდენჯერ გამეორდება ეს ასო ამ სიტყვაში, მაგრამ ამ ამოცანის გადაჭრა O(n)-ში შეგვიძლია Map-ის გამოყენებით. თუ კი Map-ში შევინახოთ თვითოუელი ასო ბგერის გამეორების სიხშირეს.

```javascript
const str = 'hello';

function countWords(str) {
    const chars = str.split('');
    const charsCount = new Map();

    for(const char of chars) {
        if(charsCount.has(char)) {
            const charValue = charsCount.get(char);
            charsCount.set(char, charValue + 1);

            continue;
        }

        charsCount.set(char, 1);
    }

    return charsCount;
}

countWords('helllooh');
// Map(4) {'h' => 2, 'e' => 1, 'l' => 3, 'o' => 2}
```
## Set
### What is Set
Set მონაცემთა სტრუქტურა ინახავს, უნიკალურ ელემენტებს რომლებსაც ერთი და იგივე მონაცემთა ტიპი გააჩნიათ, გარკვეული თანმიმდევრობით. 

### Set vs Array
Set-ი ჩვეულებრივი JS-ის მასივისგან განსხვავებით ბევრად ნელია იტერაციის, ჩაწერის და ძებნის დროს, მაგრამ მისი დანიშნულება არ არის ეს ოპერაციები, Set-ი განსაკუთრებით კარგია შემდეგი ოპერაციების დროს 

Difference

```javascript
function getDifference(a, b) {
  return new Set(
    [...a].filter(element => !b.has(element))
  );
}

const set1 = new Set(['a', 'b', 'c']);
const set2 = new Set(['a', 'b']);

console.log(getDifference(set1, set2)); // {'c'}
```

Intersection
```javascript
function getIntersection(a, b) {
  return new Set(
    [...a].filter(element => b.has(element))
  );
}

const set1 = new Set(['a', 'b', 'c']);
const set2 = new Set(['a', 'b']);

console.log(getDifference(set1, set2)); // {'a', 'b'}
```

Uniq
```javascript
const arr = ['a','a','b','b'];

console.log(new Set([...arr])) // {'a', 'b'}
```

Union
```javascript
const set1 = new Set(['a', 'b', 'c']);
const set2 = new Set(['a', 'b', 'd']);

const union = new Set([...set1, ...set2]);
console.log(union); // {'a', 'b', 'c', 'd'}
```

ამიტომ Set არ არის მასივის კონკურენტი მონაცემთა სტრუქტურა, მას თავის დანიშნულება აქვს, Set შეგვიძლია გამოვიყენოთ Lookup-ის დროს `Set.prototype.has()` მეთოდის გამოყენებით, იმ შემთხვევაში თუ მხოლოდ უნიკალური ელემენტები გვჭირდება, ასევე ყველა ზემოთ მოყვანილი შემთხვევის დროს Set-ი უფრო სწრაფია ვიდრე მასივი.

## WeakMap and WeakSet
პრაქტიკაში იშვიათად გვხვდება WeakMap და WeakSet, მაგრამ სხვადასხვა ბიბლიოთეკებში ისინი აქტიურად გამოიყენება მეხსიერების მენეჯმენტისას. რომ მივხვდეთ Weak*-ის დანიშნულებას პირველ რიგში გავიხსენოთ object reference-ბი.

#### Strong Reference 

```javascript
let obj = { name: "nikolozz" };

const people = [obj];

obj = null;

console.log(people); // [{ name: "nikolozz" }];
```

ამ შემთხვევაშ ვხედავთ რომ `people` ამყარებს ძლიერ კავშირს ობიექტთან `obj` და იმ შემთხვევაშიც თუ `obj` მივანიჭებთ მნიშვნელობას `null` რაც იმას ნიშნავს რომ Garbage Collector-მა უნდა წაშალოს ის მეხსიერებიდან, ვხედავთ რომ `people` მასივში ის ინახება ანუ მეხსიერებიდან არ იშლება.

ხოლო WeakMap, WeakSet გამოყენების დროს reference ობიექტზე არის სუსტი, ამიტომ Garbage Collector-ი მას მეხსიერებიდან ამოშლის, იმის მიუხედავად რომ მას WeakMap-ში ან WeakSet-ში ვინახავთ.

#### Weak Reference 

```javascript
let peopleMap = new WeakMap();
let person = { name: "nikolozz" };

peopleMap.set(person, true);
console.log(pets); // WeakMap{ {...} -> true }

person = null;

console.log(peopleMap); // WeakMap(0) 
```

ამ მაგალითში ვხედავთ რომ როცა `people`-ის მნიშვნელობს მივანიჭეთ `null` GC-იმ ის მეხსიერებიდან წაშალა, იმ reference-ის მიუხედავათ რაც WeakMap-ს მასზე ქონდა.

ასევე Weak* *არ არის* იტერირებადი, რადგან წინასწარ არ ვიცით იქნება თუ არა მასში ობიექტი (ის ნებისმიერ დროს შეიძლება წაიშალოს GC-ის დროს).

```javascript
const people = new WeakSet();
const nikolozz = {name: "nikolozz"};
const niko = {name: "niko"};

people.add(nikolozz);
people.add(niko);

people.has(nikolozz);    // true
people.has(niko);    // true

people.delete(nikolozz); 
people.has(nikolozz);    // false
people.has(niko);    // true
```

WeakMap-ში და WeakSet-ში key-ით მხოლოდ ობიექტები შეგვიძლია რომ გამოვიყენოთ

რომ შევაჯამოთ მთავარი განსხვავება Map და Set-გან არის Weak reference, ასევე არ შეგვიძლია ელემენტებზე იტერირება და key-ით მხოლოდ ობიექტები შეგვიძლია რომ შევინახოთ.

