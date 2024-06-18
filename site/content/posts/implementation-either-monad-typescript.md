+++
title = 'An Implementation of the Either Monad in TypeScript'
date = 2023-09-05T21:15:27+02:00
+++

Backstory to this post: after working with a bit of Kotlin, I was exposed to an interesting concept called the **Either Monad**. The concept is popular among functional programming languages, but not so much TypeScript. With my background in both, I'd like to try implement the concept in TypeScript with the goal that someone who mainly makes use of TypeScript might try it out and possibly implement it in their workflow.

I am worried that someone might see the word "Monad" and immediately run in fear as they may not fully understand the concept, because of this I would just like to say that no prior understanding monads is required for this post. Of course it would be beneficial and overall it is quite an interesting concept to know about. If you are interested, there is an [absolutely fantastic video](https://www.youtube.com/watch?v=C2w45qRc3aU) which explains the concept which I would encourage you to watch. 

The implementation of the Either Monad in Kotlin is done in a great library called [Arrow](https://apidocs.arrow-kt.io/arrow-core/arrow.core/-either/index.html). TypeScript does have similar implementations of this in a library called [fp-ts](https://gcanti.github.io/fp-ts/modules/Either.ts.html), but I'm not a major fan. I liked the implementation of Arrow so much that I've decided to try replicate it as far as possible.

*Note: the aim of this write up is to understand the inner workings of Either by building it piece by piece together. If you would like to understand how to effectively make use of it in an application, referring to some formal documentation would be best.*

## Overview
First we need to understand the value using **Either** provides, there are numerous explanations of this online which I would encourage you to read or watch but I'll give a quick overview. Either allows us to wrap values we return from functions while also providing additional context to what the value is. The context is provided using the concept of *Left* and *Right*. An error is returned inside of *Left* while success is returned inside of *Right*, this allows us to:

- Know what type the value is, think *is this value an error or a success?* We can find this out immediately by checking if its a *Left* or a *Right*.
- Handle each case, if its an error we can handle it in a certain way, if its a success we can handle it in another way. All of this is done very neatly using some helper functions we will build.

Another major value point of using Either is being able to explicitly define what a function will return, take a look at the following example:

```js 
function validate(input: string): Either<ValidationError, string> {
    ...
}
```
We immediately know that the function `validate` can either return a `ValidationError` in the case of failure, and `string` in the case of success.

Overall, it is essentially an alternative approach to the standard try catch method of exception handling you are likely already familiar with.

That was just a quick overview. If you do not fully understand so far, it is perfectly fine. This post will provide us with an implementation we can use to understand everything better through examples.

## Implementation
First we need to define a class called `Either` which can take a *Left* or *Right* value:
```js
class Either<L, R> {
  private constructor(private left: L, private right: R) {}
}
```

Then we can define functions for `left` and `right` which we can call which will initialise the class with the value. Note how we can only initialise it for with a Left OR Right value, not both. This will return an Either with a value inside of it.
```js {hl_lines=["4-6","8-10"]}
class Either<L, R> {
  private constructor(private left: L, private right: R) {}

  static right<R>(value: R): Either<any, R> {
    return new Either(null, value);
  }

  static left<L>(value: L): Either<L, any> {
    return new Either(value, null);
  }
}
```
*Note: In case you're wondering why these function are implemented a static functions, this is done because it allows us to call them from the Either namespace.*


We can already call this class like so, although we can't do much else right now:
```ts
Either.left("Some Error")         // In the case of an error
Either.right("Success")           // In the case of success
```



Lets define some helper functions which tell us if the instance of Either is a Left or a Right:

```js {hl_lines=["12-14","16-18"]}
class Either<L, R> {
  private constructor(private left: L, private right: R) {}

  static right<R>(value: R): Either<any, R> {
    return new Either(null, value);
  }

  static left<L>(value: L): Either<L, any> {
    return new Either(value, null);
  }

  isRight() {
    return this.right;
  }

  isLeft() {
    return this.left;
  }
}
```

At this point, we can already create a *Left* or *Right* **Either** instance with a value inside of it and check if its an error or a success:
```js
const result = Either.left("Error!")

console.log(result.isLeft())        // Output: true
console.log(result.isRight())       // Output: false

```


What we can't do yet is access the value inside of the Either (in this case it is `"Error"`), thats what we will do next.


We now need a function which can handle the case where the Either is a Right, this we will call `map`. 
Let's explain it as follows:
1. `map` is a function which takes another function, called *f*
2. We check to see if the Either instance is a Right, if so, we call the function *f* passing it the current Right value
3. We take the result of *f* and return it inside a Either Right instance. We return a new Either instance as it allows us to call the `map` function on the result as many times as we need, allowing us to chain. You'll see the value of this in a bit.
```js
  map<R2>(f: (wrapped: R) => R2): Either<L, R2> {
    if (this.isRight()) {
      return Either.right(f(this.right));
    }
    return Either.left(this.left);
  }
```

At this point we have the `map` function defined to handle the case where the Either is a Right.


We also need a function to handle the case where the Either is a Left. We will call this function `mapLeft`. You'll see its very similar to `map`, except rather we check to see if the Either instance is a Left instead.

```js
  mapLeft<L2>(f: (wrapped: L) => L2): Either<L2, R> {
    if (this.isLeft()) {
      return Either.left(f(this.left));
    }
    return Either.right(this.right);
  }
```


I mentioned earlier that we can chain together Either. This allows us to setup our program in such a way which resembles a pipeline, with each step neatly defined. The example below demonstrates how the value is passed down with each `map` function call.
1. We initialise an Either Right with a string, `"Success"`
2. We call the `map` function, the `map` function will check to see if the instance is a Right, if so, it will call the function passed as a parameter with the Right value.
3. We concatenate "!" onto *it* and return *it*
4. Repeat until we have "Success!!"
```js {hl_lines=[6, 9]}
const result = Either.right("Success");


result
  .map((it) => {
    return it + "!";       // We return: Success!
  })
  .map((it) => {
    return it + "!"        // We return: Success!!
  });
```

\
So far we have a basic foundation of Either setup, but unfortunately it's still not very useful. If we were to try use our implementation in a real project, we would run into some problems quite quickly. Let me define a few sample functions so I can demonstrate these problems and how we can extend our implementation to solve them.


Lets think of a simple scenario where we have an application which can handle, convert and process a request. It is structed as follows: 

1. Our main entry-point function is called `handle`, this function takes a request and will return an Either. 
2. `handle` calls another function, `convert` which will convert the request. 
3. Finally, we call the function `process` to do some form of processing, out of scope for this example.


Focusing on `handle`, note that we need to return `Either<GenericError, String>`.

```js
function handle(request: Request): Either<GenericError, string> {
  return convert(request).map((it) => process(it));
}

function convert(request: Request): Either<GenericError, Body> {
  return Either.right({ id: request.id, name: request.name });
}

function process(body: SampleBody): Either<GenericError, string> {
  // some form of processing ...
  return Either.right("success");
}
```


Because of the types we have setup so far, via static checking, TypeScript is able to tell us that the value we are trying to return inside of the `handle` function is incorrect.
![Example 1](/example1.png "700px")
Note that we are instead returning `Either<GenericError, Either<GenericError, String>>` which is incorrect. You'll see that we have somehow nested an Either inside an Either. How has this happened?

We are returning an Either from `process` which then gets returned in the Right of the Either returned from `convert`. Simply, because of the current implemetion of the `map` function, we are unable to return the correct value to satisfy the function `handle`. This occurs when we call another function which returns an Either inside the function passed to `map`.

Here is a diagram which may assist with understanding what has happened, you will see we have an Either inside the Right value of the outer Either:
![Diagram 1](/diagram1.png "500px")


To solve this, we will need to extend our implmenetation. We will add two new functions `flatMap` and `chain`.

Let's focus mainly on `chain` as it does all the hard work here, `flatMap` is merely just a wrapper for the function.


1. `chain` is a function which takes another function *f*
2. We check to see if the Either instance is a Right, if so, we call the function passing the current Right value
3. We take the result of *f* and return it

Let's discuss this for a second, notice how we simply just pass the current Right value to the function and proceed. If you scroll up and contrast this to the `map` function, you will see in that case we create a new Either instance. In this case, we simply just pass the value to the function. 
This will create a 'flattened' Either instance where the Either returned is the one from the last function call only. This solves our problem where we can end up with a nested Either.


```js
  flatMap<R2>(f: (wrapped: R) => Either<L, R2>): Either<L, R2> {
    return this.chain(f);
  }

  chain<R2>(f: (wrapped: R) => Either<L, R2>): Either<L, R2> {
    if (this.isRight()) {
      return f(this.right);
    }
    return Either.left(this.left);
  }
```

So, how does our application look now? You will see we no longer have an issue with the type returned to the `handle` function. The Either value has been flattened and now satisfies the return type.


![Example 2](/example2.png "700px")


One last useful function we will quickly add is called `catch`. This function will be based on a try catch statment. `catch` allows us to execute code which may unexpectedly throw an exception and transform it into an Either which can be handled using the functions we've implemented above.

1. `catch` is a function which takes another function *f*
2. We try call the function *f*, if it succeeds we return the value in a new Either Right
3. If we encounter an exception, we return the result in an Either Left

```js
  static catch<R2>(fn: () => R2): Either<any, R2> {
    try {
      return Either.right(fn());
    } catch (e) {
      return Either.left(e);
    }
  }
```


At this point we've implemented all the basic functions required to make use of Either sufficiently. In truth, there are many more functions which can be added here which you'll likely find if you take a look at other implementations. I feel what we've implemented is enough and provides us with a basic understanding of the concept.


As a bonus, I've added a very simple sample application below which makes use of the functions we've implemented above. Here is how it is comprised: 

1. We define some errors which are unique to each aspect of the application, in this case handling and processing
2. We define a third-party mock service which we will assume may or may not throw an exception
3. A function called `process` which calls the third party service. We make use of `Either.catch()` here because it allows us to handle any exceptions thrown from the service gracefully. We also transform any exceptions into something which makes sense, in this case `ProcessingError`
4. A function called `handle` which calls `process`. If a Either Right is returned from `process`, we log out a message to indicate it has been successful and return it. If an Either Left is returned from `process` we transform this into a suitable error which can be exposed if needed. 
```js
class ProcessingError extends Error {
  name: "ProcessingError";
}

class HandlingError extends Error {
  name: "HandlingError";
}

class Service {
  static request(input: string): string {
    return "OK";
  }
}

function process(input: string): Either<ProcessingError, string> {
  return Either.catch(() => Service.request(input)).mapLeft((it) => {
    console.log(`Unexpected error encountered: ${it}`);
    return new ProcessingError(it);
  });
}

function handle(input: string): Either<HandlingError, string> {
  return process(input)
    .map((it) => {
      console.log(`Processing succeeded, result: ${it}`);
      return it;
    })
    .mapLeft((it) => new HandlingError(it.message));
}
```

## Final Implementation
So that should be it, one last thing I will leave you with is the actual implementation we have built together which you can use if you wish. I hope you have found this post helpful and learnt something new.

```js
class Either<L, R> {
  private constructor(private left: L, private right: R) {}

  static right<R>(value: R): Either<any, R> {
    return new Either(null, value);
  }

  static left<L>(value: L): Either<L, any> {
    return new Either(value, null);
  }

  isRight() {
    return this.right;
  }

  isLeft() {
    return this.left;
  }

  map<R2>(f: (wrapped: R) => R2): Either<L, R2> {
    if (this.isRight()) {
      return Either.right(f(this.right));
    }
    return Either.left(this.left);
  }

  mapLeft<L2>(f: (wrapped: L) => L2): Either<L2, R> {
    if (this.isLeft()) {
      return Either.left(f(this.left));
    }
    return Either.right(this.right);
  }

  flatMap<R2>(f: (wrapped: R) => Either<L, R2>): Either<L, R2> {
    return this.chain(f);
  }

  chain<R2>(f: (wrapped: R) => Either<L, R2>): Either<L, R2> {
    if (this.isRight()) {
      return f(this.right);
    }
    return Either.left(this.left);
  }

  static catch<R2>(fn: () => R2): Either<any, R2> {
    try {
      return Either.right(fn());
    } catch (e) {
      return Either.left(e);
    }
  }
}
```