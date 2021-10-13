---
layout: post
title: NestJS Dynamic Modules
published: true
tags: NestJS
---

მოდულები NestJs-ში განსაზღვრავს კომპონენტებს როგორიც არის კონტროლერები და პროვაიდერები რაც აპლიკაციის მოდულარულ ნაწილებს ქმნის, ამ პოსტში დავწერ დინამიურ მოდულებზე AWS-ის მოდულის მაგალითზე.

(ეს პოსტი გულისხმობს რომ თქვენ უკვე გაქვთ [ფუნდამენტალური NestJS-ის ცოდნა](https://docs.nestjs.com/))

NestJS-ში ხშირად ვხვდებით დინამიურ მოდულებს, მაგალითად `ConfigModule` ან `TypeOrm` მათი მოდულში რეგისტრაციის დროს ჩვენ სტატიკურ მეთოდს `forRoot`-ს ვიძახებთ სადაც გადავცემთ კონფიგურაციას, `ConfigModule`-ის შემთხევავში ეს არის `path` რომელიც განსაზღვრავს `.env` ფაილის მდებარეობას, `TypeOrm`-ის შემთხვევაში ჩვენ გადავცემთ ობიექტს რომელშიც მონაცემთა ბაზასთან დასაკავშირებელი ველებია.

მაგალითად:
```typescript
@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        host: configService.get('POSTGRES_HOST'),
        port: configService.get('POSTGRES_PORT'),
        username: configService.get('POSTGRES_USER'),
        password: configService.get('POSTGRES_PASSWORD'),
        database: configService.get('POSTGRES_DB'),
        entities: [join(__dirname, '/../**/*.entity.{js,ts}')],
        synchronize: true,
      }),
      inject: [ConfigService],
    }),
  ],
})
export class DatabaseModule {}
```

დინამიურ მოდულში სტატიკურისგან განსხვავებით შეგვიძლია გადავცეთ კონფიგურაცია. ამ პოსტის მიზანია დავწეროთ დინამიური AWS მოდული, რომელსაც დავარეგისტრირებთ აპლიკაციაში და შემდეგ მისი ინექცია (Inject) შეგვეძლება ჩვენი აპლიკაციის სხვადასხვა მოდულებში სადაც AWS-ის SDK გამოიყენება.

პირველ რიგში სანამ იმპლემენტაციაზე გადავალათ უნდა ვიცოდეთ როგორ გამოვიყენებთ ამ მოდულს ჩვენს აპლიკაციაში. 

1. უნდა შეგვეძლოს AwsModule-ის Root (App) module-ში დარეგისტრირება AWS Credential-ებით.
```typescript
// src/app.module.ts
@Module({
  imports: [
    AwsModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        accessKeyId: configService.get('AWS_ACCESS_KEY_ID'),
        secretAccessKey: configService.get('AWS_SECRET_ACCESS_KEY'),
        region: configService.get('AWS_REGION'),
      }),
      inject: [ConfigService],
    })
]})
export class AppModule {}
```

2. მოდულში სადაც AWS სერვისის გამოყენება გვინდა, ვარეგისტრირებთ AwsModule იმ სერვისებს რომლის SDK-ს გამოვიყენებთ პროვაიდერებად.

```typescript
import { S3 } from 'aws-sdk';

@Module({})
export class FilesModule {
    imports: [AwsModule.forFeature([S3])]
}
```

3. Custom Provider-ს ინექციით გამოვიყენებთ ნებისმიერ პროვაიდერში

```typescript
import { S3 } from 'aws-sdk';

@Injectable()
export class FilesService {
    constructor(@InjectAwsService(S3) private readonly s3Service: S3) {}
}
```

შემდეგ s3Service შეგვიძლია გამოვიყენოთ AWS-სთან ინტერაქციისათვის, `forFeature`-ში გადაცემული credential-ები ინიცირებული იქნება AWS სერვისებისათვის.

### იმპლემენტაცია:

პირველ რიგში შევქმნათ NestJS აპლიკაცია
```bash
nest new aws-dynamic-module-example
```
და შევქმნათ მოდული რომელიც AWS-სთან ინტერაქციისთვის იქნება განკუთვნილი.
```bash
nest g mo aws
```

დინამიურ მოდულს მინიმუმ ერთი მეთოდი უნდა ქონდეს რომელიც DynamicModule-ის ინტერფეისს დააიმპლემენტირებს, `DynamicModule` უნდა აბრუნდებდეს მოდულს, ამ მოდულიდან ექსპორტირებული პროვაიდერები დაკონფიგურირებული იქნება იმ მნიშვნელობით რასაც forRoot-ში გადავცემთ.

```typescript
@Module({})
export class AwsModule {
  static forRootAsync(): DynamicModule {
    return {
      module: AwsModule,
    };
  }
}
```

imports:-თან ერთად ასევე შეგვიძლია გადავცეთ ყველა ის ველი რომელსაც სტატიკურ მოდულში გადავცემთ - providers, imports, exports, controllers.

პირველრიგში .env ფაილიდან უნდა გადავცეთ Aws credential-ები, ამისთვის დავაინსტალიროთ შემდეგი მოდულები:
```bash
npm i @nestjs/config 
```
და aws sdk
```bash
npm i aws-sdk
```

forRootAsync-ში გადავცემთ credential-ებს, რადგან მათ `ConfigService`-იდან ვიღებთ უნდა გამოვიყენოთ Custom Provider-ი.

შევქმნათ მეთოდი createProvider რომელიც პროვაიდერში დაარეგისტრირებს იმ credential-ებს რომლებსაც AwsModule.forRootAsync()-ში გადავცემთ.

```typescript
// src/aws/aws.module.ts

@Global()
@Module({})
export class AwsModule {
  static forRootAsync(options): DynamicModule {
    const providers = [this.createProvider(options)];
    return {
      module: AwsModule,
      providers,
      exports: providers,
    };
  }

  private static createProvider(options): Provider {
    return {
        provide: 'AWS_TOKEN',
        useValue: options.useFactory,
        inject: options.inject,
    }
  }
}
```

დინამიური მოდული @Global() დეკორატორით მოვნიშნეთ რადგან მასში შექმნილი ექსტპორიტერებული პროვაიდერები ყველა მოდულში ხელმისაწვდომი იყოს.

createProvider მეთოდი დაარეგისტრირებს Custom Provider-ს რომლის სახელიც იქნება 'AWS_TOKEN' და მნიშვნელობა useFactory-დან დაბრუნებული ობიექტი.

```typescript
@Module({
  imports: [
    AwsModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        accessKeyId: configService.get('AWS_ACCESS_KEY_ID'),
        secretAccessKey: configService.get('AWS_SECRET_ACCESS_KEY'),
        region: configService.get('AWS_REGION'),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

ამ შემთხვევაში useFactory-ი დააბრუნებს ობიექტს და თუ პროვაიდერს კონტეინერიდან ტოკენით ამოვიღებთ 
```typescript
app.get('AWS_TOKEN');
``` 
მივიღებთ: 
```js
{
    accessKeyId: 'ADSFFS...',
    secretAccessKey: 'ASDAF...',
    region: 'us-west-2'
}
```

*(პროვაიდერს კლასის გარდა ნებისმიერი value შეგვიძლია მივანიჭოთ - ობიქეტი, კონსტანტა)*

1. createProvider-მა დაარეგისტრირა პროვაიდერი
2. @Global() დეკორატორმა დაექსპორტებული პროვაიდერი გახადა ხელმისაწვდომი ყველა მოდულში

აპლიკაციის კონტეინერში დაინჟექტებულია პროვაიდერი შეგვიძლია ასე წარმოვიდგინოთ:
```typescript
{
  provide: 'AWS_TOKEN',
  useValue: {
    accessKeyId: 'ADSFFS...',
    secretAccessKey: 'ASDAF...',
    region: 'us-west-2'
  }
}
```

როცა პროვაიდერში AWS Credential-ები გვაქვს შეგვიძლია დავარეგისტრიროთ სერვისები ამ credential-ებით.

შევქმნათ მეთოდი რომელიც პარამეტრად მიიღებს AWS Service-ს.

```typescript
//...
private static createServiceProvider(service) {
  return {
      provide: service.serviceIdentifier // s3, dynamo, ...
      useFactory: (options: AwsConfigurationOptions) => new service(options),
      inject: ['AWS_TOKEN'] // { accessKeyId, secretAccessKey, region}
  }
}
//...
```

და ასევე სტატიკური მეთოდი რომელიც დინამიურ მოდულს დააბრუნებს:

```typescript
//...
static forFeauture(...services) {
  const providers = services.map(this.createServiceProvider);
  return {
      module: AwsModule,
      providers,
      exports: providers
  }
}
//...
```

ამ მეთოდის გამოყენებით კონტეინერში დავარეგისტრირებთ AWS Service-ებს რომლის მნიშვნელობა დაკონფიგურირებული კლასია.

შეგვიძლია კონტეინერში დაინჟექტებული პროვაიდერი ასე წარმოვიდგინოთ:
```typescript
{
    provide: 's3',
    useValue: new S3({    
      accessKeyId: 'ADSFFS...',
      secretAccessKey: 'ASDAF...',
      region: 'us-west-2'
    })
}
```

დავარეგისტრიროთ AWS მოდული იმ მოდულში სადაც AWS SDK-ს ვიყენებთ:
```typescript
@Module({
  imports: [AwsModule.forFeature(S3)],
  providers: [FilesService],
})
export class FilesModule {}
```


პროვაიდერის ინექცია შეგვიძლია @Inject დეკორატორით
```typescript
@Inject('s3') // new S3(optionsObject)
```

რადგან ჩვენ 'aws-sdk'-დან იმპორტირებულ ობიექტებს გადავცემთ forFeature-ში, შეგვიძლია დავწეროთ დეკორატორი რომელიც ამ ობიექტიდან სერვისის სახელს დააბრუნებს.

```typescript
export const InjectAwsService = (service) => service.serviceIdentifier;
```

და გამოვიყენოთ Files მოდულის პროვაიდერში (სერვისში)

```typescript
@Injectable()
export class FilesService {
  constructor(
    @InjectAwsService(S3) private s3Service: S3,
    private readonly configService: ConfigService,
  ) {}

  getBuckets() {
    return this.s3Service
      .listObjects({ Bucket: this.configService.get('AWS_BUCKET') })
      .promise();
  }
}
```

ამგვარად ერთხელ ვარეგისტრირებთ კონტეინერში AWS სერვისებს პროვაიდერებათ და Dependency Injection-ით ვიღებთ კონტეინერიდან credential-ებით ინიცირებულ სერვისებს.

კოდი შეგიძლიათ იხილოთ აქ - https://github.com/nikolozz/blog-nestjs-dynamic-module-example

