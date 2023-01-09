# Borsh TS 
[![Project license](https://img.shields.io/badge/license-Apache2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Project license](https://img.shields.io/badge/license-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![NPM version](https://img.shields.io/npm/v/@dao-xyz/borsh.svg?style=flat-square)](https://npmjs.com/@dao-xyz/borsh)
[![Size on NPM](https://img.shields.io/bundlephobia/minzip/@dao-xyz/borsh.svg?style=flat-square)](https://npmjs.com/@dao-xyz/borsh)

**Borsh TS** is a Typescript implementation of the [Borsh](https://borsh.io/) binary serialization format for TypeScript projects. The motivation behind this library is to provide more convinient methods using **field and class decorators.**

Borsh stands for _Binary Object Representation Serializer for Hashing_. It is meant to be used in security-critical projects as it prioritizes consistency,
safety, speed, and comes with a strict specification.

This implementation is performant, in fact, It slightly outperforms protobuf.js [benchmark 2](./benchmark/bench3.ts) (10% faster), [benchmark 3](./benchmark/bench3.ts) (5% faster)

### How `borsh-ts` differs from `borsh-js`
- Schema is defined using decorators rather than building a map. The schema is stored alongside the class behind the scenes so there is no longer need to pass it during serialization and deserialization. 
- Big number are interpreted with `bigint` rather than `BN` (bn.js) 
- No dependency on `Buffer` 
- ESM and CJS build
- Stricter validation checks during serialization and deserialization


## Installation

```
npm install @dao-xyz/borsh
```
or 
```
yarn add @dao-xyz/borsh
```

## Serializing and deserializing

### Serializing an object
*SomeClass* class is decorated using decorators explained later
```typescript
import {
  deserialize,
  serialize,
  field,
  variant,
  vec,
  option
} from "@dao-xyz/borsh";

class SomeClass 
{
    @field({type: 'u8'})
    x: number

    @field({type: 'u64'})
    y: bigint

    @field({type: 'string'})
    z: string

    @field({type: option(vec('u32'))})
    q?: number[]

    constructor(data: SomeClass)
    {
       Object.assign(this, data)
    }
}

...

const value = new SomeClass({ x: 255, y: 20n, z: 'abc', q: [1, 2, 3] });

// Serialize 
const serialized = serialize(value); 

// Deserialize
const deserialized = deserialize(serialized,SomeClass);
```

## Examples of schema generation using decorators
For more examples, see the [tests](./src/__tests__index.test.ts).

**Enum, with 2 variants** 

```typescript
abstract class Super {}

@variant(0)
class Enum0 extends Super {

    @field({ type: "u8" })
    public a: number;

    constructor(a: number) {
        super();
        this.a = a;
    }
}

@variant(1)
class Enum1 extends Super {

    @field({ type: "u8" })
    public b: number;

    constructor(b: number) {
        super();
        this.b = b;
    }
}

class TestStruct {

    @field({ type: Super })
    public enum: Super;

    constructor(value: Super) {
        this.enum = value;
    }
}

```
Variants can be 'number', 'number[]' (represents nested Rust Enums) or 'string' (not part of the Borsh specification). i.e.

```typescript 
@variant(0)
class ClazzA
...
@variant([0,1])
class ClazzB
...
@variant("clazz c")
class ClazzC
```


**Nested Schema generation for structs**
```typescript
class InnerStruct {

    @field({ type: 'u32' })
    public b: number;

}

class TestStruct {

    @field({ type: InnerStruct })
    public a: InnerStruct;

}
```


**Arrays**

***Dynamically sized***

```typescript
class TestStruct {
  @field({ type: Uint8Array })
  public vec: Uint8Array; 
}
```

```typescript
class TestStruct {
  @field({ type: vec('u8') })
  public vec: Uint8Array; // Uint8Array will be created if type is u8
}
```


```typescript
class TestStruct {
  @field({ type: vec('32') })
  public vec: number[];
}
```

*Custom size encoding, ***not*** compatible with Borsh specification*
```typescript
class TestStruct {
  @field({ type: vec('u64', 'u8') }) // Size will be encoded with u8 instead of u32
  public vec: bigint[]; 
}
```


***Fixed length***
```typescript
class TestStruct {
  @field({ type: fixedArray('u8', 3) }) // Fixed array of length 3
  public fixedLengthArray: Uint8Array;  // Uint8Array will be created if type is u8
}
```

```typescript
class TestStruct {
  @field({ type: fixedArray('u64', 3) }) // Fixed array of length 3
  public fixedLengthArray: bigint[];
}
```


**Option**
```typescript
class TestStruct {
  @field({ type: option('u8') })
  public a: number | undefined;
}
```

**Custom serialization and deserialization**
```typescript
class TestStruct {

    // Override ser/der of the number
    @field({
        serialize: (value: number, writer) => {
            writer.u16(value);
        },
        deserialize: (reader): number => {
            return reader.u16();
        },
    })
    public number: number;
    constructor(number: number) {
        this.number = number;
    }
}

validate(TestStruct);
const serialized = serialize(new TestStruct(3));
const deserialied = deserialize(serialized, TestStruct);
expect(deserialied.number).toEqual(3);
```


## Inheritance
Schema generation is supported if deserialization is deterministic. In other words, all classes extending some super class needs to use discriminators/variants of the same type. 

Example:
```typescript 
class A {
    @field({type: 'u8'})
    a: number 
}

@variant(0)
class B1 extends A{
    @field({type: 'u16'})
    b1: number 
}

@variant(1)
class B2 extends A{
    @field({type: 'u32'})
    b2: number 
}

```

## Discriminator
It is possible to resolve the discriminator without serializing a class completely
```typescript
import { getDiscriminator} from '@dao-xyz/borsh'

@variant([1, 2])
class A { }
class B extends A { }

@variant(3)
class C extends B { }

const discriminator = getDiscriminator(C);
expect(discriminator).toEqual(new Uint8Array([1, 2, 3]));
```

**Explicit serialization order of fields**

```typescript
class TestStruct {
    @field({ type: 'u8', index: 1 })
    public a: number;


    @field({ type: 'u8', index: 0 })
    public b: number;
}
```

This will make *b* serialized into the buffer before *a*.

## Validation

You can validate that classes have been decorated correctly: 
```typescript
validate([TestStruct])
```



## Type Mappings

| Borsh                 | TypeScript          |
|-----------------------|---------------------|
| `u8` integer          | `number`            |
| `u16` integer         | `number`            |
| `u32` integer         | `number`            |
| `u64` integer         | `bigint`            |
| `u128` integer        | `bigint`            |
| `u256` integer        | `bigint`            |
| `u512` integer        | `bigint`            |
| `f32` float           | N/A                 |
| `f64` float           | N/A                 |
| fixed-size byte array | `Uint8Array`        |
| UTF-8 string          | `string`            |
| option                | `undefined` or type |
| map                   | N/A                 |
| set                   | N/A                 |
| structs               | `any`               |

## Contributing

Install dependencies:
```bash
yarn install
```

Run tests:
```bash
yarn test
```

Run linter
```bash
yarn pretty
```

## Benchmarks 
[See benchmark script here](./benchmark/benchmar3.ts)

* json x 1,997,857 ops/sec ±0.33% (243 runs sampled)
* borsh x 3,570,224 ops/sec ±0.58% (242 runs sampled)
* protobujs x 3,357,032 ops/sec ±0.54% (241 runs sampled)

There is still some work to be done on allocating right amount of memory on serialization and improve overall performance on deserialization. JSON is fast in Javascript environments, since the JSON lib is heavily optimized in Javascript runtimes.


# License
This repository is distributed under the terms of both the MIT license and the Apache License (Version 2.0).
See [LICENSE-MIT](LICENSE-MIT.txt) and [LICENSE-APACHE](LICENSE-APACHE) for details.

For official releases see:
[Borsh]: https://borsh.io



