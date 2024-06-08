# @taktikorg/aut-fugiat

Create an async generator out of any value and pipe async generators into each other.

* [Installation](#installation)
* [from](#from)
* [combineIterable](#combineIterable)
* [combineObject](#combineObject)
* [YieldableAsyncGenerator](#YieldableAsyncGenerator)
* [ShareableAsyncGenerator](#ShareableAsyncGenerator)
* [pipe](#pipe)
* [Operators](#operators)

## Installation

```
npm i @taktikorg/aut-fugiat
```

## from

From will convert any value into an async generator.

### from using promise

Passing a promise converts it into an async generator that will yield the promise result

```js
import { from } from '@taktikorg/aut-fugiat';

const input = Promise.resolve(1);
const generator = from(input);

for await (const output of generator()) {
  console.log(output); //=> 1
}
```

### from using AsyncIterable

Passing an async iterable will yield all of its values

```js
import { from } from '@taktikorg/aut-fugiat';

async function* inputGenerator(value) {
  yield value * 2;
  yield value * 3;
  yield value * 4;
}

const input = inputGenerator(2);

const generator = from(input);

for await (const output of generator()) {
  console.log(output); //=> 4, 6, 8
}
```

### from using function 

Functions will be called with the generator arguments

```js
import { from } from '@taktikorg/aut-fugiat';

async inputGenerator(value) {
  // value will be the generator argument
  return value * 2;
}

const generator = from(inputGenerator);

for await (const output of generator(2)) {
  console.log(output); //=> 4
}
```

### from using generator function 

Generator functions will be called with the generator arguments

```js
import { from } from '@taktikorg/aut-fugiat';

async function* inputGenerator(value) {
  // value will be the generator argument
  yield value * 2;
  yield value * 3;
  yield value * 4;
}

const generator = from(inputGenerator);

for await (const output of generator(2)) {
  console.log(output); //=> 4, 6, 8
}
```

### from using a value that is non of the above

Any argument that is not a function, promise or async iterable will return a generator that will yield the passed argument.

```js
import { from } from '@taktikorg/aut-fugiat';

const input = 1;
const generator = from(input);

for await (const output of generator()) {
  console.log(output); //=> 1
}
```

## combineIterable

`combineIterable` depends on the [async-iterators-combine](https://www.npmjs.com/package/async-iterators-combine) package.
Every item in the passed iterable will be converted to an async generator (using [from](#from)) and yielded values will be combined into iterables.
For a list of all options see [the async-iterators-combine docs](https://github.com/handijk/async-iterators-combine?tab=readme-ov-file#combinelatest)

```
import { combineIterable } from '@taktikorg/aut-fugiat';

const generator = combineIterable([
  async function* (value) {
    yield value * 2;
    yield value * 3;
    yield value * 4;
  },
  (value) => Promise.resolve(value * 10),
  Promise.resolve('wow'),
  'example',
]);

for await (const output of generator(2)) {
  console.log(output); //=> [4, 20, 'wow', 'example'], [6, 20, 'wow', 'example'], [8, 20, 'wow', 'example']
}
```

## combineObject

`combineObject` behaves exactly the same as [combineIterable](#combineIterable) but accepts an object instead of an iterable and will yield objects.

```
import { combineObject } from '@taktikorg/aut-fugiat';

const generator = combineObject({
  a: async function* (value) {
    yield value * 2;
    yield value * 3;
    yield value * 4;
  },
  b: (value) => Promise.resolve(value * 10),
  c: Promise.resolve('wow'),
  d: 'example',
});

for await (const output of generator(2)) {
  console.log(output); //=> { a: 4, b: 20, c: 'wow', d: 'example' }, { a: 6, b: 20, c: 'wow', d: 'example' }, { a: 8, b: 20, c: 'wow', d: 'example' }
}
```

## YieldableAsyncGenerator

`new YieldableAsyncGenerator()` creates an object that conforms to the [async iterable and async iterator protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#the_async_iterator_and_async_iterable_protocols) and includes a `yield` method to yield values.
When a signal is passed to the constructor (`new YieldableAsyncGenerator({ signal })`) the generator will be returned as soon as the signal is aborted.

```
import { YieldableAsyncGenerator } from '@taktikorg/aut-fugiat';

const generator = new YieldableAsyncGenerator();

await Promise.all([
  (async () => {
    for await (const output of generator) {
      console.log(output) //=> 1, 2, 3
    }
  })(),
  (async () => {
    await new Promise((resolve) => setTimeout(resolve, 100));
    generator.yield(1);
    await new Promise((resolve) => setTimeout(resolve, 100));
    generator.yield(2);
    await new Promise((resolve) => setTimeout(resolve, 100));
    generator.yield(3);
  })(),
])
```

## ShareableAsyncGenerator

Extends [YieldableAsyncGenerator](#YieldableAsyncGenerator) and adds a `share` method that will return a new object that conforms to the [async iterable and async iterator protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#the_async_iterator_and_async_iterable_protocols) that will yield all values from the parent object but can be returned and thrown independently.
The `share` method also takes an object with a signal as parameter (`share({ signal })`) that will return the generator as soon as the signal is aborted.

```
import { ShareableAsyncGenerator } from '@taktikorg/aut-fugiat';

const generator = new ShareableAsyncGenerator();

await Promise.all([
  (async () => {
    for await (const output of generator.share()) {
      console.log(output) //=> 1, 2
      if (output === 2) {
        return;
      }
    }
  })(),
  (async () => {
    for await (const output of generator.share()) {
      console.log(output) //=> 1, 2, 3
    }
  })(),
  (async () => {
    for await (const output of generator.share()) {
      console.log(output) //=> 1
      if (output === 1) {
        return;
      }
    }
  })(),
  (async () => {
    await new Promise((resolve) => setTimeout(resolve, 100));
    generator.yield(1);
    await new Promise((resolve) => setTimeout(resolve, 100));
    generator.yield(2);
    await new Promise((resolve) => setTimeout(resolve, 100));
    generator.yield(3);
  })(),
])

## pipe

Will pipe the result of the first function as first argument in the following function, any arguments passed into the resulting function will be passed into the first function and as additional arguments into succeeding functions.

```js
import { pipe } from '@taktikorg/aut-fugiat';

const generator = pipe(
  async function* (value) {
    yield value * 2;
    yield value * 3;
    yield value * 4;
  },
  async function* (generator, value) {
    for await (const output of generator) {
      yield output * 1 + value;
      yield output * 2 + value;
      yield output * 3 + value;
    }
  }
)

for await (const output of generator(2)) {
  console.log(output); //=> 6, 10, 14, 8, 14, 20, 10, 18, 26
}
```

## Operators

Operators return generator functions that can be used as arguments when calling [pipe](#pipe).

### map

Map every yielded value to the value returned by the mapping method.

```js
import { pipe, map } from '@taktikorg/aut-fugiat';

async function* inputGenerator(value) {
  yield value * 2;
  yield value * 3;
  yield value * 4;
}

const generator = pipe(
  inputGenerator,
  map((value, i, originalValue) => value / 2 * i + originalValue),
);

for await (const output of generator(2)) {
  console.log(output); //=> 2, 5, 10 
}
```

### filter

Will skip the yield if the value returned by the mapping method is falsy.

```js
import { pipe, filter } from '@taktikorg/aut-fugiat';

async function* inputGenerator() {
  yield 2;
  yield 3;
  yield 4;
  yield 5;
  yield 6;
}

const generator = pipe(
  inputGenerator,
  filter((value, i, _originalValue) => value % 2 === 0),
);

for await (const output of generator()) {
  console.log(output); //=> 2, 4, 6 
}
```

### delegateMap

[Delegates](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/yield*) the async iterable to the generator function that is returned from the map method.

```js
import { pipe, delegateMap } from '@taktikorg/aut-fugiat';

async function* inputGenerator(value) {
  yield value * 2;
  yield value * 3;
  yield value * 4;
}

const generator = pipe(
  inputGenerator,
  delegateMap((value, i, _originalValue) => value % 2 === 0
    ? async function* (originalValue) {
      yield originalValue * 2;
      yield originalValue * 3;
      yield originalValue * 4;
    } : async function* (originalValue) {
      yield originalValue * 20;
      yield originalValue * 30;
      yield originalValue * 40;
    }
  )
);

for await (const output of generator(2)) {
  console.log(output); //=> 4, 6, 8, 60, 90, 120, 8, 12, 16, 100, 150, 200, 12, 18, 24
}
```

### raceMap

Uses [Promise.race](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race) to race the next yield and the map function, skips the map output and aborts the passed in [AbortSignal](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) if the next yield wins.

```js
import { pipe, raceMap } from '@taktikorg/aut-fugiat';

async function* inputGenerator(value) {
  yield value * 2;
  await new promise((resolve) => settimeout(resolve, 100));
  yield value * 3;
  await new promise((resolve) => settimeout(resolve, 50));
  yield value * 4;
}

const generator = pipe(
  inputGenerator,
  raceMap((value, i, _abortSignal, _originalValue) => {
    await new promise((resolve) => settimeout(resolve, 60));
    yield value * 10;
  })
);

for await (const output of generator(2)) {
  console.log(output); //=> 40, 80
}
```
