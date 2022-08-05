---
layout: post
title: TypeScript | Write Simple Dependency Injection
published: true
tags: JavaScript
---

Dependency Injection ("დამოკიდებულების ინექცია") ერთ-ერთი მთავარი კონცეპციაა ობიექტზე ორიენტირებულ პროგრამირებაში, რომელიც საშუალებას გვაძლევს კლასები ნაკლებად დამოკიდებულები და მარტივად ჩანაცვლებადი იყოს.


სანამ რეალიზაციაზე გადავალთ, მოკლედ დავწერ DI-ზე.

მეხუთე S.O.L.I.D-ის პრინციპის (Dependency Inversion)-ის თანახმად, კლასი უნდა იყოს დამოკიდებული აბსტრაქციაზე და არა რეალიზაციაზე (კონკრეტულ კლასზე), ამის მიღწევაში გვეხმარება DI რომელმაც იცის თუ როგორ უნდა "შექმნას" ობიექტი.

![](https://cdn-media-1.freecodecamp.org/images/1*TF-VdAgPfcD497kAW77Ukg.png)

OOP-ში ობიექტების დამოკიდებულებას ერთმანეთის მიმართ აღვნიშნავთ ტერმინებით - ასოციაცია, აგრეგაცია, კომპოზიცია და ა.შ. ამ მაგალითში მხოლოდ ამ სამ დამოკიდებულებას განვიხილავთ. მოკლედ ავღწეროთ თუ რას ნიშნავს ეს ტერმინები.

- ასოციაცია - ზოგადი ტერმინია ობიექტების დამოკიდებულების განმსაზღვრელად, მაგალითად ობიექტი a შეიცავს ობიექტ b-ს.

- აგრეგაცია - ასოციაციის ტიპია, როცა ვამბობთ რომ ობიექტ a-ს აგრეგაციაა ობიექტი b, ვგულისხმობთ რომ a ობიექტი შეიცავს b ობიექტს, მაგრამ b ობიექტი არ არის ინიციალიზებული a ობიექტის შიგნით, მათი სასიცოცხლო ციკლი არ არის ერთმანეთზე დამოკიდებული.

```typescript
class B {}

class A {
    constructor (b: B) { }
}
const b = new B();

new A(b)
```

თუ წავშლით ობიექტ a-ს, b გააგრძელებს ცხოვრებას, ობიექტები ერთმანეთთან არ არიან მჭიდრო კავშირში.

- კომპოზიცია - ყველაზე მჭიდრო კავშირია ობიექტებს შორის, ერთი ობიექტი განსაზღვრავს მეორე ობიექტის სასიცოცხლო ციკლს.

```typescript
class B { }

class A {
    private b: B;

    constructor() {
        this.b = new B();
    }
}

new A()
```

DI-ის დროს გვაქვს აგრეგაცია ობიექტებს შორის, ისინი არ არიან ერთმანეთთან მჭიდროდ დაკავშირებულები. 

რადგან შევძლოთ "დამოკიდებულებების ინექცია" გვჭირდება **კონტეინერი** სადაც შეინახება ინსტრუქცია თუ როგორ შევქმნათ ობიექტი, რა დამოკიდებულებები გააჩნია ამ ობიექტს და სხვა მეტა მონაცემები. მაგალითად [Inversify](https://inversify.io/) არის IoC (Inversion of Control) Container რომელშიც შეგვიძლია შევინახოთ ობიექტები და მათი დამოკიდებულები, ხოლო როცა ამ კლასს გამოვიყენებთ, Inversify იზრუნებს რომ მასზე დამოკიდებული ობიექტები შექმნას და მათი მნიშვნელობა გადასცეს კონსტრუქტორში.

```typescript
import { Container, injectable, inject } from "inversify";

@injectable()
class A {}

@injectable()
class B {}

@injectable()
class C {

    private a: A;
    private b: B;

    public constructor(a: A, b: B) {
        this.a = a;
        this.b = a;
    }
};


const container = new Container();
container.bind<A>(A).to(A);
container.bind<B>(B).to(B);
container.bind<C>(C).to(C);
```

მაგალითად ამ შემთხვევაში ვქმნით კონტეინერს, სადაც ვარეგისტრირებთ კლასებს, როცა C კლასს შევქმნით Inversify ამოიღებს A და B კლასებს კონტეინერიდან და შექმნის მათ instance-ებს. კლასების გარდა, შეგვიძლია ნებისმიერი მნიშვნელობის ინექცია კონტეინერში, რომელიც გარკვეული "ტოკენით" იქნება ხელმისაწვდომი.

```typescript
@injectable()
class B {
  constructor(@inject(API_KEY)) {}
}

const API_KEY = Symbol.for('API_KEY');

const container = new Container();
container.bind<string>(API_KEY).to('abc');
```

ამ შემთხვევაში `API_KEY` არის ტოკენი, რომელიც კონტეინერიდან `abc` მნიშვნელობას ამოიღებს.

Angular/Nest ფრეიმვორკების გამოყენებისას არ გვიწევს IoC კონტეინერის მენეჯმენტი, ეს ფრეიმვორკები თვითონ ამენეჯებენ კონტეინერს, მაგრამ როგორც მომხმარებლებს შესაძლებლობა გვაქვს რომ შევცვალოთ კონტეინერში არსებული ობიექტები. 

გადავიდეთ ჩვენი მარტივი DI-ის იმპლემენტაციაზე. 

როგორც ვიცით TypeScript-ში ინტერფეისი სხვა OOP ენებისგან განსხვავებით მხოლოდ დეველოპმენტის დროს გვაქვს, ხოლო კოდის ტრანსლაციის შემდეგ JavaScript-ში ინტერფეისები ქრება რადგან ის არ არის JS-ის შემადგენელი ნაწილი. 

განვიხილოთ Nest-ის მაგალითი:

```typescript
interface BasicService {
    sayHello(): void;
}

@Injectable()
class Service implements BasicService () {
    sayHello() {
        console.log('hello')
    }
}

@Controller()
class APIController {
    constructor(private readonly service: Service) {}
}
```

რა მოხდება თუ `Service`-ს `BasicService`-ით ჩავანაცვლებთ კონტსტრუქტორში? და რატომ გვიწევს ინტერფეისის მაგივრად კლასის გაწერა type-ად? 🤔

როგორც ავღნიშნეთ JavaScript–ში ინტერფეისები არ გვაქვს, ამიტომ type-ად კლასის გაწერა გვიწევს, რომელსაც გააჩნია მეტა მონაცემები, ამ მეტა მონაცემებზე დაყრდნობით Nest-ი ხვდება რომელი ობიექტი უნდა ამოიღოს კონტეინერიდან. მაგრამ საიდან ჩნდება ეს მეტა მონაცემები? 

ამის საშუალებას გვაძლევს `reflect-metadata` მისი გამოყენებით შეგვიძლია კლასს მივანიჭოთ მეტა მონაცემები რომლებსაც შემდეგში გამოვიყენებთ. 

ჩვენი ამოცანაა ქვემოთ მოცემული კოდი გადავწეროთ DI-ის გამოყენებით: 

```typescript
class WelcomeMessageGenerator {
   getMessage() {
        return `Hello! This is a worst example!`
    }
}

class App {
    constructor(private welcomeMessage: WelcomeMessageGenerator) {}

    yieldWelcomeMessage() {
        console.log(this.welcomeMessage.getMessage())
    }
}

const welcomeMessage = new WelcomeMessageGenerator();

const app = new App(welcomeMessage);
```

პირველ რიგში გვჭირდება `reflect-metadata` `package`-ი.

```sh
npm init -y && npm i reflect-metadata;
```

ასევე დაგვჭირდება დეკორატორების გამოყენება, ამიტომ `tsconfig.json`-ში დავამატოთ კონფიგურაცია:

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "target": "ES5"
  }
}
```

მაგალითად კლასის კონსტრუქტორის მეტა მონაცემების წასაკითხად შეგვიძლია გამოვიყენოთ `getMetadata`

```typescript
Reflect.getMetadata('design:paramtypes', App);
```

მაგრამ ამ შემთხვევაში დაგვიბრუნდება `undefined`, იმისთვის რომ კლასზე მეტა მონაცემები გვქონდეს საჭიროა ის იყოს დეკორირებული.


```typescript
interface ClassType<T> {
    new (...args: any[]): T;
};

function Injectable() {
    return function<T>(target: ClassType<T>) {
        console.log(Reflect.getMetadata('design:paramtypes', target))
    };
}
```

`design:paramtypes` დააბრუნებს პარამეტრებს რომლებიც კონსტრუქტორშია გადაცემული. მოვნიშნოთ კლასები `@Injectable` დეკორატორით:

```typescript
//...
@Injectable()
class WelcomeMessageGenerator {
   getMessage() {
        return `Hello! This is a worst example!`
    }
}

@Injectable()
class App {
  constructor(private welcomeMessage: WelcomeMessageGenerator) {}

  yieldWelcomeMessage() {
      console.log(this.welcomeMessage.getMessage())
  }
}
//...
```

Output:
```sh
undefined
[ [Function: WelcomeMessageGenerator] ]
```

რადგან `WelcomeMessageGenerator` კლასს კონტსტრუქტორში არ გადაეცემა პარამეტრები, დაიბეჭდება undefined პირველ ხაზზე.

ამის შემდეგ ჩვენ უკვე გაგვაჩნია მეტა მონაცემები რომ კლასებს ამ მონაცემებზე დაყრდნობით შევუქმნათ დამოკიდებულებები. 

შევქმნათ `Injector` კლასი, რომელიც პასუხისმგებელი იქნება მეტა მონაცემების მიხედვით კლასის დამოკიდებულებების შექმნაზე.

```typescript
class Injector {
  static resolve<T>(target: ClassType<T>): T {
    const tokens = Reflect.getMetadata('design:paramtypes', target);

    const tokenInstances = tokens.map(token => new token());

    return new target(...tokenInstances);
  }
}
```

ამ შემთხვევაში `resolve` სტატიკური მეთოდი, ძალიან მარტივ სამუშაოს აკეთებს

```typescript
//...
const tokens = Reflect.getMetadata('design:paramtypes', target)
//...
```

ვიღებთ კონსტრუქტორში არსებულ პარამეტრებს `[ [Function: WelcomeMessageGenerator] ]`,ვქმნით ამ კლასის instance-ბს და ვაბრუნებთ კლასს ინიცირებული დამოკიდებულებებით.

```typescript
//...
const tokenInstances = tokens.map(token => new token());

return new target(...tokenInstances);
//...
```


მაგრამ იმ შემთხვევაში თუ `WelcomeMessageGenerator` დაემატება დამოკიდებულება, მისი ინიციალიზაცია არ მოხდება, რადგან ჩვენ მხოლოდ `App` კლასის დამოკიდებულებებს ვიღებთ `tokens` ცვლადში. 

 ```typescript
//...
@Injectable()
class Greeter {
  hello() {
    return 'Hello'
  }
}

@Injectable()
class WelcomeMessageGenerator {
  constructor(private greeter: Greeter) {}

  getMessage() {
    const greeting = this.greeter.hello()
    return `${greeting}, this is a worst example!`
  }
}
//...
 ```

 Output:

 ```sh
 TypeError: Cannot read properties of undefined (reading 'hello')
 ```

რადგან ყველა კლასის დამოკიდებულების ინიციალიზაცია მოვახდინოთ, `resolve` რეკურსიულად უნდა გამოვიძახოთ, ინიცირებული კლასის instance-ით.


```typescript
//...
class Injector {
  static resolve<T>(target: ClassType<T>): T {
    const tokens = (Reflect.getMetadata('design:paramtypes', target) || []).map(
      (token) => this.resolve(token)
    );

    return new target(...tokens);
  }
}
//...
```

DI-ის გამოყენების შემდეგ, აღარ გვიწევს კლასის და მისი დამოკიდებულებების ხელით შექმნა. 


DI-ის გარეშე:
```typescript
//...
const greeter = new Greeter();
const welcomeMessageGenerator = new WelcomeMessageGenerator(greeter);
const app = new App(welcomeMessageGenerator);

app.yieldWelcomeMessage()
```

DI:
```typescript
//...
const app = Injector.resolve<App>(App);

app.yieldWelcomeMessage()
```

სრული კოდი შეგიძლიათ ნახოთ: https://github.com/nikolozz/blog-write-simple-di-example

