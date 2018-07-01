# A Deep Dive into Iterators and Generators in JavaScript

This article's a journey: The goal is to build to an understanding of generators from the ground up, detailing the intuitions behind why generators, and JavaScript generators in particular, work the way they do. Surprises along the way include iterables, symbols, coroutines, and async/await. Let's dive into square one!

## Intro to Iteration
The Google dictionary result for iteration is “the repetition of a process.”

`for` and `while` loops are both examples of iteration. Programming languages can additionally offer declarative shorthands for constructing "looped" passes through larger entities. The act of repeating an operation on an entity in a subset by subset manner can be referred to as "iterating over" that entity.

Here's an example of iteration in JavaScript using the spread operator:
```js
console.log([...'word'])
// ['w', 'o', 'r', 'd']
```
The spread operator passes over the string `'word'` character by character, performing the same operation on each. Each character is pulled from its context in the larger string and pushed onto the array on its own.

Here's another example of iteration in JavaScript:
```js
for (const char of 'word') {
  console.log(char)
}
// w o r d
```
Rather than commenting on what's going on above specifically, let's consider the general trened illustrated by this last example:
```js
for (const variable of iterable) {
  // do something
}
```
In general, iteration can be performed on what is known as an iterable; i.e., an object implementing the “iterable protocol.”

## Iteration Protocols
### Iterable protocol
According to [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#The_iterable_protocol), “The iterable protocol allows JavaScript objects to define or customize their iteration behavior.”

If you were a JavaScript object, how would you define your iteration behavior?

MDN continues: “In order to be iterable, an object...(or one of the objects up its prototype chain) must have a property with a @@iterator key which is available via [the] constant Symbol.iterator.”

Let's back up there--`Symbol`, `Symbol.iterator`, `@@iterator`, iterables--and tackle these one by one.

#### `Symbol`
In JavaScript, a `Symbol` is a primitive data type representing a unique ID token. `Symbol`s perform a whole grab bag's worth of functionality.

One thing `Symbol`s can do is expose built-in object capabilities, including the only sometimes realized ability for objects to perform iteration. That its collection of data can be iterated over is, by default, an unrealized possibility for an object literal. `Symbol`s thus expose the way in which JS objects may conform to the iterator protocol.

#### `Symbol.iterator`
One can expose certain JavaScript types' iteration behavior via accessing these types' `Symbol.iterator` property. `String`s and `Array`s both have built-in iterators (`StringIterator` and `Array Iterator`, respectively); one need only invoke a string or array's `Symbol.iterator` built-in function to access such an iterator directly. Object literals' iteration behavior must be manually defined.

In defining a `Symbol.iterator` property on an object literal, we can take a thing without default iteration behavior, set iteration behavior, and then iterate over that thing.

I.e.,
```js
const myIterable = {}
myIterable[Symbol.iterator] = ??
for (const val of myIterable) {
  // do something with val
}
```

#### `@@iterator`
“@@ describes what's called a well-known symbol” ([Stack Overflow](https://stackoverflow.com/questions/29492333/what-does-at-at-mean-in-es6-javascript)). `@@iterator` is just shorthand for `Symbol.iterator`.

#### Iterables
The `Symbol.iterator` invocation discussed can be summed up as a process of turning objects into iterables, where an iterable is an object from which you can get an iterator.

Iterables are thus objects that can be iterated over, according to the iterator protocol. The iterator protocol details how iterables work under the hood.

### Iterator protocol
In the context of the iterator protocol, [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) defines iteration as “a standard way to produce a sequence of values (either finite or infinite).” An object thus accords with the iterator protocol when
(a) it knows how to access items from a collection one at a time, while
(b) keeping track of its current position within that sequence.
An iterable is furthermore supposed to meet both these criteria in a particular way: An iterable must implement a `next()` method.

#### `next()`
`next()` is a function not requiring any arguments and returning an object with two properties, `done` and `value`.

A great analogy for how `next()` works with regard to iterables is [a Pez dispenser](https://www.youtube.com/watch?v=j1NpEIB7Yms&feature=youtu.be). You can consume the "next" candy from a Pez dispenser in a one-at-a-time fashion. When there are no more candies left, the dispenser is (at least temporarily) "done."

Let's observe `next()` in action.

First, let's set an iterable:
```js
const iterable = 'hi'[Symbol.iterator]()
```
Because the `String` data type contains the built-in `StringIterator` property, one need only invoke a string's `Symbol.iterator` built-in function to effectively set an iterable from that string. `iterable` is such a `StringIterator` object.

We can now invoke the `next()` on our iterable:
```js
iterable.next() // 1st call
// { value: "h", done: false }
```
We get back an object with the value prop set to `"h"`--the first character of `"hi"`, our original thing-to-be-iterated-over--and the done prop set to `false`.

Note that calling our iterable again, this time yielding a value of `"i"`, still keeps that done prop `false`.
```js
iterable.next() // 2nd call
// { value: "i", done: false }
```

It is *only* after we attempt to call the iterable an "*n + 1*th" time, where *n* is the number of items our iterable consumes, that we get back an object with a value of `undefined` and done set to `true`.
```js
iterable.next() // 3rd call
// { value: undefined, done: true }
```

Subsequent calls to the same iterable now yield the same result:
```js
iterable.next() // 4th call
// { value: undefined, done: true }
iterable.next() // 5th call
// { value: undefined, done: true }
...
iterable.next() // nth call
// { value: undefined, done: true }
```

Note that if we are, say, streaming in data, *n* may be very large or even infinite. It is thus conceivable for an iterable not to terminate.

## Intro to Generators
We've glossed how to take a thing without default iteration behavior, set iteration behavior via [Symbol.iterator], and then iterate over that thing:
```js
const iterableObj = {}
iterableObj[Symbol.iterator] = ?????
for (const val of iterableObj) {
  // something with val
}
```
But what actually happens on that second line? What should you set an iterable-to-be's `Symbol.iterator` property to?

A generator, of course!
```js
iterableObj[Symbol.iterator] = INSERT GENERATOR HERE
```

We now have our requirements for a generator function: **A generator function must return an iterable object, of a kind with you would get from invoking `Symbol.iterator`; and this iterable "generator object" must conform to iterator protocol, supporting a next() method that returns an object with value and done props.**

Our work's cut out for us!

### Generator Syntax
Let's make a generator:
```js
const myIterable = {}

myIterable[Symbol.iterator] = function*() {
  let i = 3
  while (i > 0) yield i--
  yield 'liftoff!'
}

for (const val of myIterable) console.log(val)
/* logs the following:
3
2
1
liftoff!
*/
```
What's going on above? We're able to use `myIterable` as an iterable by defining its `Symbol.iterator` property to be a generator. `myIterable` then communicates (somehow) with the generator available at `myIterable[Symbol.iterator]` to yield the results we expect.

Meanwhile, we employ two significant bits of syntax in our generator:

#### `function*`
The "*" in the function signature signifies a generator function.
#### `yield`
The `yield` keyword functions as a “play/pause” button for generators. Generator execution can be paused and resumed at yield statements.

Generators thus supports the iterable protocol in addition to the iteration protocol by allowing for the creation of a pausable, resumable--"yieldable"--function. Iteration is performed in the `for...of` loop via the yielding of next values.

We can condense the code block above to the following:
```js
const myGenerator = function*() {
  let i = 3
  while (i > 0) yield i--
  yield 'liftoff!'
}
for (const val of myGenerator()) console.log(val)
/* logs the same:
3
2
1
liftoff!
*/
```

#### Invoking a Generator
Let's dispel some magic. What happens when we invoke the `myGenerator` function?
(1) A new iterable "generator object" is created: `const generatorObj = {}`. (You cannot use the “new” keyword to construct an iterable generator object. Instead, you must create a generator function and invoke it.)
(2) A `Symbol.iterator` property is instantiated on the new iterable generator object, and set to the `myGenerator` function: `generatorObj[Symbol.iterator] = myGenerator`.
(3) The iterable generator object is returned.

Altogether, it works out like this:
```js
const myIterable = myGenerator()
--------------------------------
const generatorObj = {}
generatorObj[Symbol.iterator] = myGenerator
return generatorObj
--------------------------------
const myIterable = generatorObj
```
We can now iterate through the generator through use of the iterator protocol's `next()` method. The `for...of` loop does this implicitly.

### Usage of Generators
Generators can be used for a variety of problems, including but not limited to executing asynchronous operations (think fetch and database calls) and lazily-loading data.

If you're familiar with `async/await` syntax, consider that generators can be used for any operations for which you’d use async/await. As noted [here--L, “Generators, Coroutines, Async/Await” (highly recommended)--](https://www.wptutor.io/web/js/generators-coroutines-async-javascript)Babel will actually transpile async/await syntax to generator syntax. As the just-cited article notes, this is because “async functions are just syntactic sugar on top of generators. They use the exact same concept as coroutines.”

Which, of course, leads to the question of what a coroutine actually is.

#### Generators and Coroutines
As defined in the above article, “Coroutines are a programming concept that allows functions to pause themselves and give control to another function. The functions pass control back and forth.” We need some way to manage generators, and coroutines constitute this level of management.

Control flow goes something like this:
```js
Coroutine                           Generator
---------                           ---------
hey
                                    hey
done?
                                    nah
etc.
```

##### Example: Let's Seed a Database
Let's say we're seeding a DB. We've got some dummy data and want to call a single `seed()` method to perform operations including adding the single user and all the moves to our DB, as well as, say, instantiating a one-many relation between user and moves. (For the sake of this example, I'll be using the ORM syntax of [Sequelize](http://docs.sequelizejs.com/manual/) for instance creation.)
```js
const hedgehogUser = {
  id: 1, name: ‘Mittens’, age: 5
}
const hedgehogMoves = [
  { id: 1, description: ‘ball up’ },
  { id: 2, description: ‘roll’ },
  { id: 3, description: ‘go to work’ },
  { id: 4, description: ‘charlie brown’ },
  { id: 5, description: ‘cha cha real smooth’ }
]

seed()
```
We could, and probably should, write `seed()` using async/await syntax...
```js
async function seed() {
  const [ user, moves ] = await Promise.all([
    Hedgehog.create(hedgehogUser),
    Move.bulkCreate(hedgehogMoves)
  ])
  await user.setMoves(moves)
}
```

...or, we could write `seed()` as a coroutine managing a generator in the following way:
```js
function* seedGenerator() {
  const [ user, moves ] = yield Promise.all([
    Hedgehog.create(hedgehogUser),
    Move.bulkCreate(hedgehogMoves)
  ])
  yield user.setMoves(moves)
}

function seedCoroutine(generator) {
  const iterable = generator()
  function doNext(data) {
    const { value, done } = iterable.next(data)
    if (!done) value.then(doNext)
  }
  doNext()
}

function seed() {
  return seedCoroutine(seedGenerator)
}
```
###### A Note on `next(args)`
Before we dive into the control flow, let's address the `next(data)` call above. `next()` *can* take in an argument. In this way, our *generator* can access the argument passed through `next()`. When we thus resume the generator, we resume *right where we left off*, and whatever we pass to `next()` gets passed into the generator as the value of the `yield` statement where we left off. This way, if we've assigned some variable to this `yield` statement, this variable can hold the value of whatever we passed into whatever `next()` statement got us back to the `yield`.

###### Control Flow Walkthrough
The generator/coroutine combo makes for the following sequence of events when calling `seed()`. (I'd encourage you to read over the following list primarily to get a sense of the general pattern, and only secondarily to try and understand how every line of code is working in each and every context.)
(1) The first thing that happens when we call `seed()` is that `seedCoroutine` is called with our `seedGenerator` passed in as an argument.
(2) `seedGenerator` is invoked, giving us an iterable object, `iterable`, whose `Symbol.iterator` property has been set to `seedGenerator`.
(3) Still within `seedCoroutine`, the `doNext` function gets invoked with no arguments.
(4) Within `doNext`, `data === undefined`. So when we call `next(data)` off `iterable`, we're effectively calling `next(undefined)`.
(5) `seedCoroutine` "pauses" at `next()` and hands off control to `seedGenerator`.
(6) `seedGenerator` runs, yielding an array of promises for a `hedgehogUser` instance and `hedgehogMoves` instances, respectively.
(7) `seedGenerator` "pauses" at `yield` and hands off control to `seedCoroutine`.
(8) `seedCoroutine` resumes right where it left off, at `next()`. `iterable.next(data)` becomes `{ value: Promise, done: false }`. We destructure the `value` and `done` properties off of this object.
(9) `done === false`, so we wait until `value` settles. (We might also synchronously return a promise at this point to any outer scope requiring knowledge of what our coroutine is up to. This is how async/await works: an async function returns a promise that can eventually resolve to the async function's eventual return value.)
(10) When `value` settles, we call `doNext`, passing in the resolved data as an argument: `doNext(resolvedData)`. In this case, the resolved data is an array of a `hedgehogUser` object and an array of `hedgehogMoves` objects.
(11) We then call `iterable.next(resolvedData)`.
(12) `seedCoroutine` "pauses" at `next()` and hands off control to `seedGenerator`.
(13) `seedGenerator` resumes right where it left off, at `yield`. The entire yield statement (`yield Promise.all ...`) becomes `resolvedData`. We destructure `[ users, moves ]` from the `resolvedData` array.
(14) `seedGenerator` continues execution. We reach the next `yield` statement, yielding a promise for the user on whom we want to set some moves.
(15) `seedGenerator` "pauses" at `yield` and hands off control to `seedCoroutine`.
(16) `seedCoroutine` resumes at `next()`. `iterable.next(previouslyResolvedData)` becomes `{ value: Promise, done: false }`. We destructure the `value` and `done` properties off of this object.
(17) `done === false`, so we wait until `value` settles.
(18) When `value` settles, we call `doNext`, passing in the resolved data as an argument: `doNext(newlyResolvedData)`. In this case, the resolved data is a single `hedgehogUser` instance.
(19) We then call `iterable.next(newlyResolvedData)`.
(20) `seedCoroutine` "pauses" at `next()` and hands off control to `seedGenerator`.
(21) `seedGenerator` resumes at the same `yield` statement at which it left off. The entire yield statement (`yield user.setMoves(moves)`) becomes `newlyResolvedData`.
(22) We hit the end of the `seedGenerator` function. By default, `seedGenerator` hands off control to `seedCoroutine`, passing back an object with `value: undefined` and `done: true`.
(23) `seedCoroutine` resumes at `next()`. `iterable.next(data)` becomes `{ value: undefined, done: true }`. We destructure the `value` and `done` properties off of this object.
(24) `done !== false`, so we don't have to wait until `value` settles. We're done!

We have just seen how coroutines can manage generators to support custom iteration behavior. As the author of the by-now-clearly-inspiration-article-for-this-article, “Generators, Coroutines, Async/Await” notes of this use case:
> As long as the generator only `yield`s promises, `next.value` will always be a promise. The coroutine calls then on the promise to run `doNext` again when the promise finishes. When `doNext` runs again, it unpauses the generator and gets the next promise. This repeats until there’s no more `yield`s in the generator. **Each time coroutine calls `next`, control passes to the generator. Each time the generator calls `yield`, control returns to coroutine.** This concept of passing control back and forth asynchronously is known as coroutine. You can use synchronous control flow with coroutines.

## Conclusion
At this point you've hopefully gained some familiarity with the generator interface. There are several key uses for generators besides seeding a database. Generators can be used to compose middleware, coordinate fetch requests, implement streaming (lazy loading, infinite scrolloing) behavior--basically any operations for which you'd use async/await. As L, author of “Generators, Coroutines, Async/Await” notes, "Async functions are just syntactic sugar on top of generators. They use the exact same concept as coroutines." In fact, [Babel](https://babeljs.io/docs/en/babel-plugin-transform-async-to-generator/) will actually transpile async functions to generators-plus-coroutines. As L concludes, "Even though async functions and coroutines are essentially the same thing, async/await is a lot more intuitive." Go forth and use async/await! You can now hopefully better reason about what's going on under the hood.
