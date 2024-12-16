---
layout: post
title: NodeJS | Optimizing NodeJS Applications with Go and WebAssembly
published: true
tags: NodeJS, Go, WebAssembly
---

Golang გამოიყენება CPU-ინტენსიური ამოცანებისთვის და გარკვეული დავალებებისთვის მას უკეთესი შედეგის მოცემა შეუძლია, ვიდრე JavaScript-ს, რომელიც NodeJS-ის გარემოში სრულდება.

განვიხილოთ შემდეგი მაგალითები:

- გამომთვლელი ოპერაციები, რომლებიც შეიძლება მოიცავდეს nested loop-ს, დიდი რიცხვების გამრავლებას ან სხვა ოპერაციებს, რომლებიც დიდ პროცესორის რესურსს მოითხოვს
- თუ გვაქვს მიკროსერვისები Go-ზე და Node-ზე და გვსურს ამ მიკროსერვისებისთვის გარკვეული კოდის გაზიარება, ორი სხვადასხვა იმპლემენტაციის დაწერის შემთხვევაში ვერ ვიქნებით დარწმუნებულები, რომ ისინი ერთნაირად შესრულდებიან

ზემოთ მოცემულ სიტუაციებში უმჯობესია იმპლემენტაცია Golang-ზე დავწეროთ და შემდეგ გამოვიყენოთ NodeJS-ში. Node-ში შეგვიძლია გავუშვათ C++-ზე დაწერილი კოდი, რომელიც ბაინდინგებით უკავშირდება, რასაც Go-სთვის ვერ გავაკეთებთ. ამისთვის დაგვჭირდება გარდამავალი ენა, რომელშიც შეგვიძლია Go დავაკომპილიროთ და Node-ში გავუშვათ. სათაურიდან უკვე მიხვდებოდით, რომ საუბარია Webassembly-ზე. ამ სტატიაში WASM-ზე ვრცლად არ დავწერ. უბრალოდ უნდა ვიცოდეთ, რომ Webassembly არის ბინარული ფორმატის, low-level პროგრამირების ენა. Rust/C/C++/Golang შეიძლება დაკომპილირდეს WASM-ში, რომელიც შეგვიძლია გავუშვათ ბრაუზერში.

მაგალითისთვის ავიღოთ ალგორითმი ერატოსთენეს საცერი, რომელიც კრიპტოგრაფიაში გამოიყენება. ეს ალგორითმი დააბრუნებს მასივიდან ყველა მარტივ რიცხვს, რომელიც ≤n.

```js
function sieveOfEratosthenes(n) {
    const isPrime = new Array(n + 1).fill(true);
    isPrime[0] = isPrime[1] = false;

    for (let p = 2; p * p <= n; p++) {
        if (isPrime[p]) {
            for (let i = p * p; i <= n; i += p) {
                isPrime[i] = false;
            }
        }
    }

    const primes = [];
    for (let p = 2; p <= n; p++) {
        if (isPrime[p]) {
            primes.push(p);
        }
    }

    return primes;
}
```

და ანალოგიურად მისი იმპლემენტაცია Golang-ზე:

```go
func sieveOfEratosthenes(n int)[] int {
    isPrime := make([] bool, n + 1)
    for i := range isPrime {
        isPrime[i] = true
    }

    for p := 2; p * p <= n; p++ {
        if isPrime[p] {
            for i := p * p; i <= n; i += p {
                isPrime[i] = false
            }
        }
    }

    var primes[] int
    for p := 2; p <= n; p++ {
        if isPrime[p] {
            primes = append(primes, p)
        }
    }
 
    return primes
}
```

ახლა კი შევადაროთ, რა სისწრაფით სრულდება ეს ალგორითმები ამ ორი იმპლემენტაციისთვის.

![](https://i.imgur.com/TjWSMtT.png)

ამ ბენჩმარკზე ვხედავთ, რომ Go უფრო სწრაფად ასრულებს ამ ალგორითმს. ეს უპირატესობა კიდევ უფრო იზრდება, როცა n = 1e8, სადაც Go ამ ალგორითმს 532 მილიწამში ასრულებს, ხოლო Node - 38.12 წამში.

თუ Node-ში გვინდა Go-ზე დაწერილი `sieveOfEratosthenes`-ის გამოყენება, პირველ რიგში დავაკომპილიროთ Golang WASM-ში.

შემდეგი ბრძანება შექმნის `.wasm` ფაილს ბინარული კოდით:

```bash
GOARCH=wasm GOOS=js go build -o soe.wasm 
```

მაგრამ სამწუხაროდ ეს საკმარისი არ არის იმისთვის, რომ ეს ბინარული კოდი Node-ში გამოვიყენოთ.

1. დავამატოთ `main` ფუნქცია, რომელიც განუწყვეტლივ შესრულდება
2. დავამატოთ `binding`-ები, რომელიც ერთგვარი ხიდი იქნება Go-სა და JavaScript-ს შორის
3. დავაკომპილიროთ კოდი WASM-ში
4. დავაექსპორტოთ `wasm_exec.js` სკრიპტი, რომელიც შექმნის runtime გარემოს Go-სთვის და მოგვცემს საჭირო ფუნქციებს NodeJS-ში WASM-ის კოდის გასაშვებად

პირველ რიგში შევქმნათ `main` ფუნქცია, რომელიც განუწყვეტლივ იმუშავებს, სანამ პროცესი აქტიურია:

```go
func main() {
    <-make(chan bool)
}
```

გამოვიყენეთ channel, რომელიც main-ს სამუდამოდ შეასრულებს.

გადავიდეთ ყველაზე საინტერესო ნაწილზე: [syscall/js](https://pkg.go.dev/syscall/js)-ის გამოყენებით წვდომა გვექნება JS-ზე. მისი საშუალებით შეგვიძლია შევქმნათ ფუნქცია, რომელსაც JS-ი შეძლებს გამოიყენოს:

```go
func sieveWrapper() js.Func {
    return js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        if len(args) != 1 {
            return "Invalid number of arguments passed"
        }

        n := args[0].Int()
        result := sieveOfEratosthenes(n)

        jsArray := js.Global().Get("Array").New(len(result))
        for i, v := range result {
            jsArray.SetIndex(i, v)
        }

        return jsArray
    })
}
```

გავარჩიოთ ზემოთ მოცემული კოდი: `js.FuncOf` აბრუნებს ფუნქციას, რომელიც შეიძლება გამოძახებულ იქნას JS-დან. პირველი პარამეტრი იქნება `this`, ხოლო მეორე პარამეტრში კი ყველა დანარჩენი JS-დან გადაცემული არგუმენტი ჩაიწერება.

`n := args[0].Int()` აიღებს პირველ გადაცემულ არგუმენტს (რადგან მხოლოდ `n`-ს გადავცემთ `sieveOfEratosthenes` ფუნქციას) და გამოიძახებს `sieveOfEratosthenes`-ს ამ პარამეტრით.

`js.Global().Get("Array").New(len(result))` ქმნის ახალ JS მასივს, სადაც `SetIndex` მეთოდი ჩაწერს ინდექსის მიხედვით მნიშვნელობებს.

ამ ფუნქციის ექსპორტისთვის დაგვჭირდება მისი `Global`-ში დარეგისტრირება. ამისთვის `main` ფუნქციაში დავამატოთ:

```go
js.Global().Set("sieveOfEratosthenes", sieveWrapper())
```

`sieveWrapper` აბრუნებს `js.Func` ტიპს, რომელიც გამოიძახება JS-ში sieveOfEratosthenes-ის გამოძახების დროს.

სრული კოდი ასე გამოიყურება:

```go
//go:build wasm

package main

import (
    "fmt"
    "syscall/js"
)

func sieveOfEratosthenes(n int) []int {
    // ...
}

func sieveWrapper() js.Func {
    return js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        if len(args) != 1 {
            return "Invalid number of arguments passed"
        }

        n := args[0].Int()
        result := sieveOfEratosthenes(n)

        jsArray := js.Global().Get("Array").New(len(result))
        for i, v := range result {
            jsArray.SetIndex(i, v)
        }

        return jsArray
    })
}

func main() {
    fmt.Println("Hello Webassembly!")

    js.Global().Set("sieveOfEratosthenes", sieveWrapper())

    // Keep the Go program running
    <-make(chan bool)
}
```

`//go:build wasm` build tag-ი საჭიროა დავამატოთ, რათა კომპილატორს ვუთხრათ, რომ მხოლოდ wasm-ის კომპილაციის დროს დააკომპილიროს ეს ფაილი.

ხელახლა დავაგენერიროთ .wasm ფაილი:

```bash
GOARCH=wasm GOOS=js go build -o soe.wasm 
```

ასევე, როგორც მეოთხე პუნქტში დავწერეთ, დაგვჭირდება `wasm_exec.js` ფაილი, რომლის ექსპორტირება შეგვიძლია ბრძანებით:

```bash
cp "$(go env GOROOT)/misc/wasm/wasm_exec.js" .
```

გადავიდეთ Node-ის ნაწილზე. ამ .wasm კოდის გასაშვებად დაგვჭირდება მისი ინიციალიზაცია, რაც შემდეგნაირად შეგვიძლია:

```js
async function loadWasmModule() {
    const go = new Go();
    const wasmBuffer = await readFile(join(__dirname, "soe.wasm"));
    const {
        instance
    } = await WebAssembly.instantiate(
        wasmBuffer,
        go.importObject
    );

    go.run(instance);

    return {
        sieveOfEratosthenes: globalThis.sieveOfEratosthenes,
    };
}
```

`wasm_exec.js` ქმნის გლობალურ ცვლადს `Go`, რომელიც ახორციელებს კომუნიკაციას Go-სა და JS-ს შორის. go.importObject-ის საშუალებით აწვდის ყველა ფუნქციას, რომელიც Go-ს ჭირდება JS-თან საკომუნიკაციოდ.

`go.run(instance)` შეასრულებს `Go`-ს პროგრამას WASM-ში. ამ ფუნქციის გამოძახების შემდეგ გაეშვება `main` ფუნქცია, რომელიც ზემოთ შევქმენით, დაილოგება `Hello, Webassembly!` და `sieveOfEratosthenes` ფუნქცია ჩაიწერება `global`-ში.

სრული კოდი გამოიყურება ასე:

```js
const {
    readFile
} = require("fs/promises");
const {
    join
} = require("path");
require("./wasm_exec");

async function loadWasmModule() {
    const go = new Go();
    const wasmBuffer = await readFile(join(__dirname, "soe.wasm"));
    const {
        instance
    } = await WebAssembly.instantiate(
        wasmBuffer,
        go.importObject
    );

    go.run(instance);

    return {
        sieveOfEratosthenes: globalThis.sieveOfEratosthenes,
    };
}

async function main() {
    try {
        const {
            sieveOfEratosthenes
        } = await loadWasmModule();

        const primes = sieveOfEratosthenes(100);
        console.log("Primes up to 100:", primes);
    } catch (err) {
        console.error("Error:", err);
    }
}

main();
```

Output:
```bash
Hello Webassembly!
Primes up to 100: [
   2,  3,  5,  7, 11, 13, 17, 19,
  23, 29, 31, 37, 41, 43, 47, 53,
  59, 61, 67, 71, 73, 79, 83, 89,
  97
]
```

მოდით, ხელახლა ვნახოთ, რა იქნება ამ ფუნქციის შესრულების დრო Go-ში, NodeJS-სა და NodeJS + Webassembly გარემოში n = 1e8-სთვის.

![](https://i.imgur.com/dD6yBAK.png)

როგორც ვხედავთ, NodeJS + WASM დაახლოებით 16-ჯერ უფრო სწრაფად სრულდება, ვიდრე JS-ის იმპლემენტაცია.