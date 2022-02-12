---
layout: post
title: TypeScript | Utility Types
published: true
tags: TypeScript
---

TypeScript-ის სტანდარტულ ბიბლიოთეკაში არის უტილიტები რომლებიც გვეხმარება ტიპების გარდაქმნაში.
ამ პოსტში დავწერ იმ უტილიტებზე რომლებსაც ყველაზე ხშირად ვიყენებთ პროექტებში ასევე გაჩვენებთ მათ იმპლემენტაციას.

მაგალითად ავიღოთ შემდეგი კოდი რომლის მაგალითზეც გამოვიყენებთ უტილიტებს:

```typescript
interface IPost {
  id: number
  title: string
  content: string
}

class Post {
  constructor(public post: IPost) {}
}

const savePost = (post: Post) => {}

const editPost = ({ title, content }: IPost) => {}
```

### Partial

ხშირად ინტერფეისიდან არ გვჭირდება ყველა `property`

```typescript
const editPost = ({ title, content }: IPost) => {
  const post = new Post({ title, content })
}
```

ამ შემთხვევაში ჩვენ გვინდა რომ მხოლოდ `title` და `content` განვაახლოთ, მაგრამ TypeScript-ი არ დაკოპმილირდება:

```
Property 'id' is missing in type '{ content: string; title: string; }' but required in type 'IPost'.
```

რადგან ინტერფეისში აღწერილი ყველა ტიპი არ არის გადაცემული (TypeScript-ში `?` გარეშე ყველა `property` _აუცილებელია_)

შეგვიძლია გამოვიყენოთ `Partial` `Generic Type Utility` რომელიც ყველა ველს ინტერფეისზე ოპციონალურს გახდის.

```typescript
class Post {
  constructor(public post: Partial<IPost>) {}
}
```

`Partial`-ი ინტერფეისს შემდეგნაირად გარდაქმნის

```typescript
interface IPost {
  id?: number | undefined
  title?: string | undefined
  content?: string | undefined
}
```

(რეალურ პროექტზე `create` და `update` ოპერაციებისთვის სხვადასხვა ინტერფეისები უნდა იყოს გამოყენებული, ხშირად `Type` შეცდომები არასწორ არქიტექტურაზე მიუთითებს)

ასეთი `generic`-ის დასაწერად საჭიროა `keyof`-ის და `extend`-ის და `in` ოპერატორის გამოყენება.

#### keyof

აბრუნებს ინტერფეისის ველებს:

```typescript
type IPostKeys = keyof IPost
// id | title | content
```

#### extend

ავრცელებს ინტერფეისს

```typescript
type DetailedPost = IPost & { createdAt: Date }
/*
{
  id: number;
  title: string;
  content: string;
  createdAt: Date
}
*/
```

დავწეროთ Partial-ის იმპლემენტაცია `extend და keyof გამოყენებით:

```typescript
type MyPartial<T> = {
  [P in keyof T]?: T[P]
}
```

`MyPartial` `Generic` ტიპის უტილიტაა რომელიც პარამეტრად იღებს ტიპს (ინტერფეის) და ანიჭებს `?` რომელიც გარდაქმნის ველს ოპციონალურად

```typescript
[P in keyof T]?: T[P]
```

კონსტრუქცია შეგვიძლია შემდეგნაირად წარმოვიდგინოთ

```typescript
type MyPartial<IPost> = {
  /*id?: IPost['id']*/
  id?: number
  /*title?: IPost['title']*/
  title?: string
  /*content?: IPost['content']*/
  content?: string
}
```

`in keyof T` აბრუნებს `id | title | content` -> `P` ამ შემთხვევაში არის `id` (პირველი დაბრუნებული ველი) ვხდით ოპციონალურს `?` და ვანიჭებთ ტიპს `: T[P]` როგორც JavaScript ობიექტში ისე: `<T>` ამ შემთხვევაში `IPost`-ია `IPost['id']` დააბრუნებს id ველის ტიპს `number`-ს.

ამ ცოდნით შეგვიძლია სხვა `Generic Type` უტილიტებზე და მათ იმპლემენტაციაზე გადასვლა.

### Required

`Required` უტილიტი ყველა ველს აუცილებელ ველად გარდაქმნის, მისი იმპლემენტაცია ძალიან გავს Partial-ის იმპლემენტაციას, იმ განსხვავებით რომ `Partial`-ში `?` ველს ოპციონალურს ვხდით ხოლო `readonly` ველს აუცილებელ ველად გარდაქმნის.

```typescript
class Post {
  constructor(public post: Required<IPost>) {}
}
```

იმპლემენტაცია:

```typescript
type MyRequired<T> = {
  [P in keyof T]-?: T[P]
}
```

შეგვიძლია კოდი შემდეგნაირად გადავწეროთ:

```typescript
const savePost = (post: Required<IPost>) => {}

const editPost = ({ title, content }: Partial<IPost>) => {}
```

`create`-ის დროს ყველა ველი აუცილებელი იქნება, ხოლო `update`-ის დროს კი ოპციონალური.

### Readonly

Readonly უტილიტი არ გვაძლევს საშუალებას ობიქეტზე ველს სხვა მნიშვნელობა მივანიჭოთ.

```typescript
const create = (post: Readonly<Post>) => {
  post.title = `${new Date()}-${post.tile}`
}

/*Cannot assign to 'title' because it is a read-only property.*/
```

იმპლემენტაცია:

```typescript
type MyReadonly<T> = {
  readonly [P in keyof T]: T[P]
}
```

როგორც წინა შემთხვევებში ვიტერირებთ ინტერფეისის ველებზე და ვანიჭებთ `readonly`-ს.

### Record

`Record Generic`-ი იღებს ორ პარამეტრს `<K, T>` ველებს და ტიპს რომელიც ამ ველებს მიენიჭება.

```typescript
type IPostString = Record<keyof IPost, string>
// type IPostString = Record<'id' | 'title' | 'content', string>
```

იმპლემენტაცია:

```typescript
type MyRecord<K extends string | number, T> {
    [P in K]: T
}
```

### Pick

`Pick Generic` უტილიტი გარდაქმნის ინტერფეის იმ ველებად რომელიც მას პარამეტრად გადავეცით, ის იღებს ორ პარამეტრს `<T, K>` სადაც `T` ინტერფეისია (ტიპი) და `K` ველების გაერთიანება (union)

```typescript
type PostBody = Pick<IPost, 'title' | 'content'>
/*
{
  title: string;
  content: string;
}
*/
```

იმპლემენტაცია:

```typescript
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P]
}
```

`K extends keyof T` აბრუნენს `T` ტიპში (ინტერფეისში) შემავალ ველებს, ანუ ერთგვარი შეზღუდვა არის რომ სხვა ველი არ გადავცეთ მაგალითად: `created` რომელიც ამ შემთხვევაში `IPost`-ის შემადგენელი ველი არ არის.

`[P in K]: T[P]` კონსტრუქციით განვსაზღვრავთ ველს და მის ტიპს.

ეს კონსტრუქცია შეგვიძლია ასე წარმოვიდგინოთ:

```typescript
type MyPick<IPost, K extends 'id' | 'title' | 'content'> = {
    /*[P in K]: T[P]*/
    id: number;
    title: string'
}
```

### Omit

`Omit` უტილიტი `Pick`-სგან განსხვავებით ინტერფეისიდან შლის ველს რომლის გაერთიანებასაც (union) პარამეტრად გადავცემთ

```typescript
type PostBody = Omit<IPost, 'id'>
/*
{
  title: string;
  content: string;
}
*/
```

ამ შემთხვევაში `id` არ იქნება `PostBody`-ის ტიპზე.

იმპლემენტაცია:

```typescript
type MyOmit<T, K extends string> = {
  [P in Exclude<keyof T, K>]: T[P]
}
```

[დანარჩენი უტილიტები და ოფიციალური დოკუმენტაცია](https://www.typescriptlang.org/docs/handbook/utility-types.html)
