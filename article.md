JavaScript is currently the most popular programming language according to many platforms (such as GitHub). However, does popularity equate to being the most advanced or beloved language? It lacks certain constructs that are considered integral parts of other languages, such as an extensive standard library, immutability, and macros. But there is one detail that, in my opinion, doesn't receive enough attention - generators.

In this article I want to explain possible use cases for iterators and generators and how they improve the verbosity of one's code. I hope, that after reading this article the following piece of code will make all the sense:

```javascript 
while (true) {
    const data = yield getNextChunk();
    const processed = processData(data);
    try {
        yield sendProcessedData(processed);
        showOkResult();
    } catch (err) {
        showError();
    }
}
```

This is the first part of a series: Iterators and generators.

## Iterators
So, an iterator is an interface that provides sequential access to data.

As you can see, the definition doesn't mention anything about data structures or memory. Indeed, a sequence of null values can be represented as an iterator without occupying memory space.

Let's have a few examples:

Array is probably a first thing that comes to mind when you think about iterators. It's a data structure that stores a sequence of values in memory. It's also an iterator, because it provides sequential access to its elements.

```javascript
const arr = [1, 2, 3];
for (const item of arr) {
    console.log(item);
}
```

The same goes for strings. They are stored in memory as a sequence of characters and provide sequential access to them.

```javascript
const str = 'abc';
for (const char of str) {
    console.log(char);
}
```

Let's look at the following function example:

```javascript
const fn = () => Math.random();
```

This function can be considered an iterator, because it provides sequential access to random numbers.

Then why do we need iterators if arrays, one of the basic data structures in the language, allow us to work with data both sequentially and in arbitrary order?

Let's imagine that we need an iterator that implements a sequence of natural numbers or Fibonacci numbers or any other infinite sequence. It would be difficult to store an infinite sequence in an array. We would need a mechanism for gradually populating the array with data and removing old data to prevent filling up the entire memory of the process. This unnecessary complexity adds extra implementation and maintenance overhead, whereas a solution without an array can be achieved in just a few lines of code:

```javascript
const getNaturalRow = () => {
    let current = 0;
    return () => ++current;
};
```


Iterators can also be used to represent data retrieved from an external channel, such as a WebSocket.

In JavaScript, any object that has a next() method, which returns a structure with value (the current iterator value) and done (a flag indicating the end of the sequence), is considered an iterator. This convention is described in the [ECMAScript language standard](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-iteration). Such an object implements the Iterator interface. Let's rewrite the previous example in this format:

```javascript
const getNaturalRow = () => {
    let current = 0;
    return {
        next: () => ({ value: ++current, done: false }),
    };
};
```


In JavaScript, there is also the Iterable interface. It represents an object that has a @@iterator method (accessible via Symbol.iterator constant) that returns an iterator. Objects that implement this interface can be iterated using the for..of loop. Let's rewrite our example once again, this time as an Iterable implementation:

```javascript
const naturalRowIterator = {
    [Symbol.iterator]: () => ({
        _current: 0,
        next() { return {
            value: ++this._current,
            done: this._current > 3,
       }},
   }),
}

for (num of naturalRowIterator) {
    console.log(num);
}
// output: 1, 2, 3
```

As you can see, we had to make flag "done" to change at some point, otherwise the loop would be infinite.

## Generators
The next stage in the evolution of iterators was the introduction of generators. They provide syntactic sugar that allows values of an iterator to be returned as if they were the result of a function. A generator is a function declared with an asterisk `function*` and returns an iterator. The iterator itself is not explicitly returned; instead, the function yields values of the iterator using the `yield` keyword. When the function completes its execution, the iterator is considered done (subsequent `next` method calls will return `{ done: true, value: undefined }`.

```javascript
function* naturalRowGenerator() {
    let current = 1;
    while (current <= 3) {
        yield current;
        current++;
    }
}

for (num of naturalRowGenerator()) {
    console.log(num);
}
// output: 1, 2, 3
```

Even in this simple example, the main nuance of generators is apparent: the code inside a generator function **does not execute synchronously**. The execution of a generator code occurs incrementally, as a result of `next` method calls on the corresponding iterator. Let's examine how the generator code executes using the previous example. We will use a special cursor to mark where the execution of the generator is paused.

At the moment of calling naturalRowGenerator, an iterator is created.

```javascript
function* naturalRowGenerator() {
    ▷let current = 1;
    while (current <= 3) {
        yield current;
        current++;
    }
}
```

Next, when we call the `next` method three times, or in our case, iterate through the loop three times, the cursor is positioned after the yield statement.

```javascript
function* naturalRowGenerator() {
    let current = 1;
    while (current <= 3) {
        yield current; ▷
        current++;
    }
}
```

And for all subsequent `next` calls, as well as after exiting the loop, the generator completes its execution. The result of subsequent `next` calls will be `{ value: undefined, done: true }`.


## Passing parameters to iterators
Let's imagine that we need to add the functionality to reset the current counter and start counting from the beginning in our iterator of natural numbers.

```javascript
naturalRowIterator.next() // 1
naturalRowIterator.next() // 2
naturalRowIterator.next(true) // 1
naturalRowIterator.next() // 2
```

It's clear how to handle such a parameter in a custom iterator, but what about generators?
It turns out that generators support parameter passing!

```javascript
function* naturalRowGenerator() {
    let current = 1;
    while (true) {
        const reset = yield current;
        if (reset) {
          current = 1;
        } else {
          current++;
        }
    }
}
```

The passed parameter becomes available as the result of the yield operator. Let's try to clarify this using the cursor approach. At the moment of creating the iterator, nothing has changed. Now let's stop after the first `next` method call:

```javascript
function* naturalRowGenerator() {
    let current = 1;
    while (true) {
        const reset = ▷yield current;
        if (reset) {
          current = 1;
        } else {
          current++;
        }
    }
}
```

The cursor is positioned after returning from the yield operator. With the next `next` call, the value passed into the function will set the value of the `reset` variable. But what happens to the value passed in the very first `next` call? It goes nowhere! If you need to pass an initial value to the generator, you can do so through the generator's arguments. Here's an example:

```javascript
function* naturalRowGenerator(start = 1) {
    let current = start;
    while (true) {
        const reset = yield current;
        if (reset) {
          current = start;
        } else {
          current++;
        }
    }
}

const iterator = naturalRowGenerator(10);
iterator.next() // 10
iterator.next() // 11
iterator.next(true) // 10
```

## Conclusion
We have explored the concept of iterators and their implementation in JavaScript. Additionally, we have learned about generators, a syntactic construct for conveniently implementing iterators.

Although in this article, I provided examples with numeric sequences, iterators in JavaScript can solve a wide range of tasks. They can represent any data sequence and even many finite state machines. In the next article, I would like to discuss how generators can be used to build asynchronous processes (coroutines, goroutines, CSP, etc.).
