---
layout: post
title: NodeJS | AsyncLocalStorage and UnitOfWork Pattern – Context Management and Transaction Handling
published: true
tags: NodeJS, Design Patterns
---

NodeJS-ის ერთ-ერთი საინტერესო API არის `AsyncLocalStorage`, რომელიც საშუალებას გვაძლევს, შევინახოთ და წავიკითხოთ მონაცემები კოდის შესრულების პროცესში.

ამ სტატიაში განვიხილავთ `AsyncLocalStorage`-ის გამოყენების ძირითად პრინციპებს, პრაქტიკულ მაგალითებს, მის API-ს, ასევე დავაიმპლემენტირებთ ლოგერს რომელიც დალოგავს trace-id-ს და საშუალებას მოგვცემს გავიგოთ რომელ request-ს ეკუთვნის ლოგი.

შემდეგ განვიხილოთ პატერნი Unit Of Work (UoW) რომელიც გვაძლევს საშუალებას შევასრულოთ ტრანზაქციები ისე რომ ბიზნეს ლოგიკაში ORM-ის ან სხვა კონკრეტული მონაცემთა ბაზის დეტალები არ გამოვიყენოთ.

### AsyncLocalStorage API

NodeJS-ში `AsyncLocalStorage`-ი იყენებს `async_hooks` მოდულს, რაც მას ასინქრონულ კონტექსტზე აძლევს წვდომას. `AsyncLocalStorage`-ის გამოსაყენებლად საჭიროა შევქმნათ მისი instance და გადავცეთ ფუნქცია რომელიც მის კონტექსტში შესრულდება.

```typescript
import { AsyncLocalStorage } from 'node:async_hooks';

const als = new AsyncLocalStorage<Map<string, string>>();
const store = new Map([['trace-id', '1']]);

als.run(store, () => {
  const traceId = als.getStore()?.get('trace-id');
  console.log(traceId); // 1
});
```

განვიხილოთ ზემოთ მოყვანილი კოდი:
1. შევქმენით ობიექტი `als`, რომელიც ინახავს კონტექსტს (store) `Map` მონაცემთა სტრუქტურაში.
2. `als.run`: ქმნის ასინქრონულ კონტექსტს, რომელში გადაცემულ ფუნქციას აქვს წვდომა პირველ არგუმენტათ გადაცემულ store-ზე (ჩვენს შემთხვევაში `new Map([['trace-id', '1']])`).
3. `als.getStore`: აბრუნებს იმ მონაცემებს, რომლებიც `run` ფუნქციის კონტექსტში ჩაიწერა.

`als.run` ქმნის კონტექსტს მხოლოდ მიმდინარე ოპერაციების ჯაჭვისთვის, ასევე ჩვენ შეგვიძლია ამ store-ის ცვლილება, მაგალითად:

```typescript
als.run(store, () => {
  const traceId = als.getStore()?.get('trace-id');
  console.log(traceId); // 1
  
  als.getStore()?.set('trace-id', 2);
  console.log(traceId); // 2
});

`run`-ს შეუძლია ნებისმიერი მონაცემთა ტიპი მიიღოს კონტექსტათ, იქნება ეს `string`,`number`,`boolean`,`object` თუ სხვა.
```

**[ვრცელი დოკუმენტაცია](https://nodejs.org/api/async_context.html#class-asynclocalstorage)**

### Logger with AsyncLocalStorage

წარმოვიდგინოთ, რომ ჩვენს სერვისში გვჭირდება ლოგერის ბიბლიოთეკის დაწერა რომელიც გამოიტანს `trace-id`-ს ყველა ლოგში.

ამ მაგალითისთვის ლოგერს მარტივი ინტერფეისით ავღწერთ, რომელსაც საკმაოდ მწირი ფუნქციონალი აქვს. 

```typescript
type LogFn = (message: string) => void;

interface Logger {
  debug: LogFn;
  info: LogFn;
  // warn, error, critical...
}

class DummyLogger implements Logger {
  public debug(message: string): void {
    console.log(`${new Date().toISOString()} - ${message}`);
  }

  public info(message: string): void {
    console.log(`${new Date().toISOString()} - ${message}`);
  }
}
```

ამ ლოგერში მხოლოდ დრო და შეტყობინება ლოგირდება. თუმცა, მოთხოვნაა, რომ ლოგებში დავამატოთ `trace-id`, ყველაზე მარტივი გზაა `trace-id` დამატებით პარამეტრად გადავცეთ, მაგრამ ამ შემთხვევაში ინტერფეისის შეცვლა მოგვიწევს, რაც კიდევ ბევრ ცვლილებას გამოიწვევს კოდში:

```typescript
type LogFn = (message: string, traceId?: string) => void;
```

ამ შემთხვევაში შეგვიძლია გამოვიყენოთ `AsyncLocalStorage` სადაც შევინახავთ `trace-id`-ს.

```typescript
class DummyLogger implements Logger {
  constructor(private readonly als: AsyncLocalStorage<Map<string, string>>) {}

  public debug(message: string): void {
    console.log(`${this.getTraceId()} ${new Date().toISOString()} - ${message}`);
  }

  public info(message: string): void {
    console.log(`${this.getTraceId()} ${new Date().toISOString()} - ${message}`);
  }

  private getTraceId(): string {
    return this.als.getStore()?.get('trace-id') || '';
  }
}
```

თუ `trace-id` შენახულია `AsyncLocalStorage`-ში ამოვიღებთ მას და ჩავწერთ ლოგში, ყოველგვარი ინტერფეისის ცვლილების გარეშე.

რადგან კონტექსტიდან ვიღებთ` trace-id`-ს საჭიროა მისი ამ კონტექსტში ჩაწერა, ამისთვის შევქმნათ HTTP სერვერი რომელიც მიიღებს request-ებს, ამოიღებს `trace-id`-ს ჰედერიდან და შეინახავს მას `AsyncLocalStorage`-ში. 

```typescript
// ლოგერი იგივე რჩება
const storage = new AsyncLocalStorage<Map<string, string>>();
const logger = new DummyLogger(storage);

const getTasksUsecase = () => {
  logger.info('Fetching tasks');
  return { id: 1, description: 'buy milk', done: false };
};

const app = express();

app.get('/tasks', (req, res) => {
  const store = new Map<string, string>();
  const traceId = req.headers['x-trace-id']?.toString() || uuidv4();
  
  store.set('trace-id', traceId);
  
  storage.run(store, () => {
    logger.info('Handling GET /tasks request');
    const result = getTasksUsecase();
    res.json(result);
  });
});

app.listen(3000);
```
გავუშვებთ შემდეგ request-ი:
```bash
curl -H "X-Trace-Id: 12345" http://localhost:3000/tasks 
```

და შევამოწმოთ ლოგი:
```bash
12345 2024-11-09T08:54:19.690Z - Handling GET /tasks request
12345 2024-11-09T08:54:19.692Z - Fetching tasks
```
როგორც ვხედავთ `trace-id` ლოგში ზუსტად ის არის რაც request-ის დროს გადავეცით, ამგვარად შეგივძლია ვნახოთ რომელი ლოგი რომელ request-ს ეკუთვნის, რაც გაგვიმარტივებს debugging-ს.

### Unit Of Work with AsyncLocalStorage 
სანამ შემდეგ მაგალითზე გადავალთ, განვიხილოთ, რა არის Unit Of Work დიზაინ პატერნი. ალბათ ხშირად შეგხვედრიათ მსგავსი კოდი:

```typescript
class OrdersService {
  private queryRunner: QueryRunner;

  constructor(queryRunner: QueryRunner) {
    this.queryRunner = queryRunner;
  }

  async createOrder(orderData: any): Promise<void> {
    await this.queryRunner.startTransaction();

    try {
      await this.queryRunner.manager.insert('orders', orderData);
      await this.queryRunner.manager.update(
        'inventory',
        { productId: orderData.productId },
        { quantity: () => 'quantity - 1' }
      );

      await this.queryRunner.commitTransaction();
    } catch (error) {
      await this.queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await this.queryRunner.release();
    }
  }
}
```

ამ კოდის მთავარი პრობლემა არის ზედმეტად მჭიდრო კავშირი ORM-თან ბიზნეს ფენაზე. სწორედ ამ პრობლემის გადასაჭრელად გამოიყენება Unit Of Work დიზაინ პატერნი, რომელიც ერთგვარი აბსტრაქციაა რეპოზიტორებზე. ([იხილეთ Repository Pattern](https://www.geeksforgeeks.org/repository-design-pattern/))

![](https://asp.net/media/2578149/Windows-Live-Writer_8c4963ba1fa3_CE3B_Repository_pattern_diagram_1df790d3-bdf2-4c11-9098-946ddd9cd884.png)

ეს პატერნი .NET-ის ეკოსისტემაში აქტიურად გამოიყენება. ვეცადოთ, რომ მოვარგოთ NodeJS-ს. ჩვენი მთავარი ამოცანაა მონაცემთა ბაზისგან აბსტრაგირება ბიზნეს ფენაზე, იქნება ეს სერვისი თუ use case-ი. 

ამ მაგალითისთვის მონაცემთა ბაზად ავიღოთ MongoDB, სადაც შესაძლებელია ტრანზაქციების გამოყენება. რა თქმა უნდა, ეს პატერნი ნებისმიერ მონაცემთა ბაზას ან ORM-ს შეგვიძლია მოვარგოთ, სადაც შეგვიძლია ტრანზაქციების შესრულება.

პირველ რიგში, ავღწეროთ Unit Of Work ინტერფეისი. ამ პატერნის სხვადასხვა იმპლემენტაციები არსებობს, სადაც მას აქვს პირდაპირი წვდომა რეპოზიტორებზე, მეთოდები `startTransaction`, `commit`, `rollback` და სხვა. შეგიძლიათ მოიძიოთ სხვა იმპლემენტაციებიც, ამ მაგალითისთვის მას ექნება მხოლოდ ერთი მეთოდი `withTransaction`, რომელიც შეასრულებს ფუნქციას ტრანზაქციის ფარგლებში:

```typescript
export interface UnitOfWork {
  withTransaction<T>(work: () => Promise<T>): Promise<T>;
}
```

სანამ ამ პატერნის იმპლემენტაციაზე გადავალთ, განვიხილოთ MongoDB-ის ტრანზაქციები. 

MongoDB-ის ტრანზაქციის მართვა ხდება სესიის საშუალებით. პირველ რიგში, უნდა შევქმნათ სესია. ტრანზაქციის დასაწყებად ვიძახებთ სესიაზე `startTransaction()`-ს. წარმატების შემთხვევაში ცვლილებები ფიქსირდება `commitTransaction()`-ით, ხოლო შეცდომის შემთხვევაში გაუქმდება `abortTransaction()`-ით.

ყოველ გამოძახებულ MongoDB ოპერაციას სესია უნდა გადავცეთ:

```typescript
const session = client.startSession();
session.startTransaction();

const usersCol = client.db('mydb1').collection('users');
const ordersCol = client.db('mydb2').collection('orders');

await usersCol.insertOne({ username: "nik" }, { session });
await ordersCol.insertOne({ orderId: 1, user: 1 }, { session });

await session.commitTransaction();
await session.endSession();
```

რადგან გავიგეთ, როგორ მუშაობს MongoDB-ის ტრანზაქციები ([ნახეთ დოკუმენტაცია ვრცელი ინფორმაციისათვის](https://www.mongodb.com/docs/manual/core/transactions/)), გადავიდეთ Unit Of Work-ის კონკრეტულ იმპლემენტაციაზე: 

```typescript
export class MongooseUnitOfWork implements UnitOfWork {
  private session: ClientSession | null = null;

  constructor(
    private readonly connection: Connection,
    private readonly als: AsyncLocalStorage<Map<'session', ClientSession | null>>
  ) {}

  public async withTransaction<T>(work: () => Promise<T>): Promise<T> {
    await this.startTransaction();
    this.als.getStore()?.set('session', this.session);

    try {
      const result = await work();
      await this.commit();
      return result;
    } catch (error) {
      await this.rollback();
      throw error;
    }
  }

  private async startTransaction(): Promise<void> {
    if (this.session) {
      throw new Error('Transaction already started');
    }

    this.session = await this.connection.startSession();
    this.session.startTransaction();
  }

  private async commit(): Promise<void> {
    if (!this.session) {
      throw new Error('No active transaction');
    }

    await this.session.commitTransaction();
    await this.session.endSession();
    this.session = null;
  }

  private async rollback(): Promise<void> {
    if (!this.session) {
      throw new Error('No active transaction');
    }

    await this.session.abortTransaction();
    await this.session.endSession();
    this.session = null;
  }
}
```

განვიხილოთ ეს კოდი დეტალურად:

- `withTransaction`: მთავარი მეთოდია, რომელიც იღებს ასინქრონულ ფუნქციას (`work`) და ამ ფუნქციის შესრულებას აკონტროლებს ტრანზაქციის ფარგლებში. თუ ფუნქცია წარმატებით შესრუდა, ცვლილებები ფიქსირდება (`commit`), ხოლო შეცდომის შემთხვევაში გაუქმდება (`rollback`). 
- `startTransaction`: იწყებს ახალ ტრანზაქციას MongoDB-ის სესიის გამოყენებით.
- `commit`: ინახავს ცვლილებებს მონაცემთა ბაზაში, თუ ფუნქცია წარმატებით სრულდება.
- `rollback`: აუქმებს მიმდინარე ტრანზაქციას შეცდომის შემთხვევაში.

ასევე, მნიშვნელოვანია, რომ სესია შევინახოთ `AsyncLocalStorage`-ში, რათა იგი ხელმისაწვდომი იყოს რეპოზიტორებისთვის. 

***Note***: საჭიროა, რომ `MongooseUnitOfWork` კლასი ყოველ request-ზე ახალი შევქმნათ, რათა თითოეულ მოთხოვნას ჰქონდეს უნიკალური სესია, `NestJS`-ის შემთხვევაში შეგიძლიათ Injection Scope გამოიყენოთ (`@Injectable({scope: Scope.REQUEST})`).

ახლა დავწეროთ რეპოზიტორი, რომელიც გამოიყენებს `AsyncLocalStorage`-ს სესიის მისაღებად:

```typescript
export class OrderRepository {
  constructor(
    private readonly orderModel: Model<Order>,
    private readonly als: AsyncLocalStorage<Map<'session', ClientSession | null>>
  ) {}

  async createOrder(orderData: Partial<Order>): Promise<Order> {
    const session = this.als.getStore()?.get('session');
    const order = new this.orderModel(orderData);
    return await order.save({ session });
  }
}
```

`createOrder` ბაზაში ქმნის შეკვეთას, სესიის გამოყენებით, რაც უზრუნველყოფს ტრანზაქციის უსაფრთხოებას.

***Note***: AsyncLocalStorage-ის ინსტანსი `OrderRepository`-თვის და `MongooseUnitOfWork`-თვის ერთი და იგივე უნდა იყოს. 

გადავწეროთ `OrdersService` წინა მაგალითიდან ამ პატერნის გამოყენებით:
```typescript
class OrdersService {
  constructor(
    private readonly unitOfWork: UnitOfWork,
    private readonly orderRepository: OrderRepository,
    private readonly inventoryRepository: InventoryRepository
  ) {}

  async createOrder(orderData: any): Promise<void> {
    await this.unitOfWork.withTransaction(async () => {
      await this.orderRepository.createOrder(orderData);
      // InventoryRepository იყენებს AsyncLocalStorage სესიის ასაღებად.
      await this.inventoryRepository.updateInventory(orderData.productId, -1);
    });
  }
}
```

თუ `updateInventory` დაფეილდა, ამ შემთხვევაში ორდერი არ შეიქმნება.


Unit Of Work პატერნი და `AsyncLocalStorage` საშუალებას გვაძლევს, მოვახდინოთ ბიზნეს ლოგიკისგან მონაცემთა ბაზის აბსტრაგირება, რაც მნიშვნელოვნად აუმჯობესებს კოდის მართვადობასა და მოქნილობას.

წყაროები:
 - https://learn.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application