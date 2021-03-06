# angular-http-deserializer
Angular Http Deserializer

Want to get started?  Head straight to [Usage](#usage).

The current Angular Http Client, since it uses typescript, uses duck typing for returning objects.  i.e. The object being returned is not an instance of the model, it's a json object emulating the shape of the model object.

Therefore, any getters, methods, etc. on the model object being returned will not exist and instanceof will not work.  This library was built specifically in order to deserialize deep object models and provide a way of easily returning constructed instances of the model objects from the Angular Http Client.

# Problem Statement

Without the use of this module, the objects being created by the angular http client are not **instances of the model**.

```javascript
showConfig() {
  this.configService.getConfig()
    .subscribe((data: Config) => {
        if (!(data instanceof Config)) {
            throw new Error('data is not instance of Config.');
        }
    });
}
```

A dialog of the issue [can be seen here](https://github.com/angular/angular/issues/20770).

# Wrong or Partial Answers

Some solutions on SO include:

* [Object.assign After Construction](https://stackoverflow.com/questions/50452431/angular-6-httpclient-return-instance-of-class#answer-50469920)

This solution is not simple enough and requires you to write constructors for each object in order for your object to be deserialized.

Deeply nested (including array) objects are not handled with this solution without more custom constructor code.

This solution has a number of pitfalls.  The Date data type is entirely missed because the incoming Json type will be string or number and will be assigned as such on the constructed object, thereby overwriting the property with the wrong type, which fail silently because of typescript's duck typing.

[Fail](https://jsfiddle.net/windhandelimprov/upkrd4bj/3/): 
```javascript
class Cow {
  sound: string;
  createdDate: Date;
}

let cow: Cow = Object.assign(new Cow(), {
    createdDate: '1/1/2018 12:00pm'
});
// Fails
expect(cow.createdDate instanceOf Date).toBeTruth();
```

* [Object.setPrototypeOf](https://stackoverflow.com/questions/35573527/angular-2-map-http-response-to-instance-of-class#answer-43790188)

This has the previously mentioned issues, along with being [very poor performance as described here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf).

* [Input Parameters Constructor](https://stackoverflow.com/questions/49499941/cant-map-angular5-httpclient-responses-to-my-typescript-classes#answer-49500145)

All of previously mentioned issues.

This solution you have to write static createInstance, a constructor code and it has the aforementioned pitfalls.

# Usage

The following are the required prerequisites:

1. [Use TypeScript](https://www.typescriptlang.org/docs/handbook/angular.html)
2. [Installation](#installation)
3. [Configuration Completed](#configuration)
4. [Model Annotation](#model-annotation)
5. [Converters](#Converters)
6. [Http Client Deserializer Injection](#http-client-deserializer-injection)
7. [Test Updates](#test-updates)
8. [Expected Exceptions](#expected-exceptions)
9. [Example Angular Project](#example-project)

# Installation

`npm install angular-http-deserializer --save`

# Configuration

In order to utilize and emit the necessary @dataType annotation you have to setup typescript within [tsconfig.json](https://www.typescriptlang.org/docs/handbook/compiler-options.html).

```
{
    ...
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
}
```

# Model Annotation

This library uses a minimalistic model annotation methodology in order to reduce the amount of effort involved in annotating the model object.  This was one of the downsides of existing deserialization libraries on npm.

You only have to provide a @dataType annotation on the types Date, Object or properties that require converters.

```typescript
import { dataType, skip, converters } from 'angular-http-deserializer/decorators';

export class User {
    id: number;
    @skip()
    name: string;

    @dataType(Date)
    @converters({
        'number': (input: number) => {
            let convertedDate = new Date(input);
            // Used for validation of the converter.
            convertedDate.setMilliseconds(123);
            return convertedDate;
        }
    })
    createdDate: Date;
    
    get wasCreatedFirstOfMonth() : boolean {
        return this.createdDate.getDate() == 1;
    }
}

export class Product {
    id: number;
    name: string;

    get hasName(): boolean {
        return this.name && this.name.length > 0;
    }
}

export class OrderProduct {

    @dataType(Product)
    product: Product;

    quantity: number;
}

export class Order {
    id: number;

    @dataType(OrderProduct, true) // Second parameter indicates isArray
    products: OrderProduct[];

    @dataType(User)
    orderedBy: User;

    @dataType(Date)
    createdDate: Date;
}
```

#Converters

Converters are created based on an object with a key of the expected incoming type and returning the expected output type for that property.  *All expected input types must have converters or an exception is thrown.*

```typescript
export class ConverterOrder {
    id: number;
    @dataType(Complex.OrderProduct, true)
    products: Complex.OrderProduct[];
    @dataType(Complex.User)
    orderedBy: Complex.User;
    @dataType(Date)
    @converters({
        'number': (input: number) => {
            let convertedDate = new Date(input);
            // Used for validation of the converter.
            convertedDate.setMilliseconds(suffixedMs);
            return convertedDate;
        }
    })
    createdDate: Date;
}
```

The above example shows a converter defined for the createdDate property with an incoming data type of number, which creates a Date type.

If you do not cover all of the potential incoming data types into the converter, **an Error will be thrown**.

# Http Client Deserialization Injection

You'll need to first import the deserializer into your service.

```typescript
import deserializer from 'angular-http-deserializer';
```

*NOTE:* Take note of the ending **r**.  Deserialize**r** creates a deserialization function for you (productino a deserialize function).  Deserialize does the deserialization itself.

From the angular http client examples:

```javascript
showConfig() {
  this.configService.getConfig()
    // Deserializer function provided to map function.
    .map(deserializer<Config>(Config))
    .subscribe((data: Config) => this.config = {
        heroesUrl: data['heroesUrl'],
        textfile:  data['textfile']
    });
}
```

Without the mapping and proper deserialization, the objects coming out of the Http Client will fail instanceof checks.

# Test Updates

Since your view can now expect non-duck typed objects (*real*) you'll likely want to change around some test code.  You'll want to take your regular json objects within your tests and convert them into real objects.  There's two ways you can do that, you can construct the normal deserializer as you would within a service that uses the Http Client or you can use the deepDeserializer.  This is an example of how this would work.

```typescript
import { deserialize } from 'angular-http-deserializer';

let productQtyJson  {
    product: null,
    quantity: 14112
};
let productQty: ProductQuantity = deserialize<ProductQuantity>(ProductQuantity, productQtyJson);
```

This way, within your tests the proper objects will now be available and property getters and methods will be accessible.

# Expected Exceptions

The deserializer is built fairly resiliently so that most things pass.  Currently, no custom deserialization is built into the annotation to enable overriding how deserialization works.  Ya get what ya get.

There are 6 expected exceptions within this module. They are the following:

## Missing necessary dataType annotation

Reason: When a property is an object, but offers no @dataType annotation to deserialize object.
Message: DataType annotation missing on Type ${type.prototype.constructor.name} field ${key}

## Expected array

Reason: When a data annotation is marked as array, but the data is not an array.
Messages:
  Array deserialization error. ${type.prototype.constructor.name}.${key} must be array.
  Array deserialization error. Object must be array.

## Array not expected

Reason: When a data annotation is **not** marked as array, but the data is.
Message: ${type.prototype.constructor.name}.${key} array not expected.

## Invalid Date cast type

Reason: Dates may be cast from 2 types, string and number.  Null or undefined are simply returned.
 If an unexpected data type is found, an exception is thrown. 
Message: Date cannot be cast from type ${expectedType}

## Missing required converter

Reason: When a converter has been supplied, but not **all** of the necessary converters.  When one is supplied, you have to cover all of the necessary cases for conversion.
Mesage: Converters for property type ${propTypeString} required.

## Skip and Converters applied to same property

Reason: 
Message: Converters cannot be skipped for property ${propertyName}.

# Example Project

[Here is a sample project commit](https://github.com/windhandel/angular-6-registration-login-example/commit/c6875c1ed6c077935568a25f6d0347e3b53445cd?diff=split) if you'd like to view the changes necessary to implement the angular-http-deserializer.

You'll want to [review the updates to the method signature](https://github.com/windhandel/angular-6-registration-login-example/commit/d5cc92a199909586b191b07bc21e76f3f41d9cfe) commit as well.