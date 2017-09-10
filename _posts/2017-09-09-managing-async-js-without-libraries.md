---
layout: post
title:  "Managing async JavaScript without libraries"
date:   2017-09-09 10:45:58 -0700
categories: javascript async
---

Generally, when developing I use libraries to handle common use cases in my code. Most developers do this. As good a s libraries are, they can obfuscate some what happens behind the covers. I like to know what is happening. To that end I have created a thought piece to explore how one might manage some aspects of async code.

I have sample code in my [async experiments repo](https://github.com/jpaulptr/async-experiments).

## Basic Callback
By async I mean some piece of code that is called some time after the main thread has executed. At its simplest, a function called in a timeout is an example of async code. The function passed to the timeout function will be fired at sometime after the containing function has completed.

{% highlight js %}
  setTimeout(() => {
    console.log('something happened')
  }, 100)  
{% endhighlight %}

## Using Async Callbacks in Serial
If this were the extent of a async methods in JavaScript, I wouldn't have much to write about. However, what happens if you want to make more than one async calls in serial, so that the next call is only made after the previous completes? This is a common scenario when one async call relies on the result of the first. 

You could write something like this, and while it is serial execution in the sense that each method is invoked after the other, it doesn't guarantee that the next will fire only after the previous has completed. Instead, what will happen is they will complete in seeming random order. Worse, if you want to pass the result of one function to another or handle errors, you can't.

{% highlight js %}
    // naive implementation
    for (let i = 0; i < max; i++) {
      arrayOfFunctions[i]();
    }
{% endhighlight %}

What is needed is a way to manage the async execution. In the code chained runner example, I have a series of functions that return an array with the results placed in an array in order. There are 3 functions to demonstrate how this works:
* getData which takes a number to use during it's calculation and a callback to call after it has completed processing. It also calls set timeout before it invokes the callback to that it becomes an async method.
* getDataChained which binds a value that the getData method will use. This is intended to simulate the result of a data call, perhaps a call to a database. Once it has set up the functions, it passes them to a method which will process the functions.
* Finally, we have the core controller, chainedRunner. It has the responsibility of managing each function call. I will explain how it works after the jump.

{% highlight js %}
const getData = (key, callback) => {

    const multiplier = parseInt(Math.random() * 10) * key;

    setTimeout(() => {
      callback(key + key);
    }, multiplier * 50);
  },

const getDataChained = function (key, callback) {
    const funcs = [];

    for (let i = 0; i < key; i++) {
      funcs.push(this.getData.bind(undefined, i + 1));
    }

    chainedRunner(funcs, callback);
  },
 

const chainedRunner = (funcs, callback) => {

  //results store
  const results = [];
  //index of current iteration
  let index = 0;

  //wrap the callback and iterate 
  //through the list of functions in order
  const wrapper = (value) => {

    index++;
    results.push(value);

    if (index < funcs.length) {
      funcs[index](wrapper);
    } else {
      //When you get to the end of the array of functions,
      //call the original callback
      callback(null, results);
    }
  };

  //start the initial call
  funcs[index](wrapper);
};
{% endhighlight %}

chainedRunner has two parts: the results store and index, to keep track of the iteration; and a wrapper function that is called recursively each time the previously invoked. Looking closely at the  wrapper function, each time it is called 
1. it receives the result of the most recent calculation
2. adds it to the results array
3. if there are functions remaining to all, it calls the next function passing the wrapper function. Remember, that the functions that are called have their data bound and only expect a callback.
4. once all the functions are completed, the callback passed in by the user is invoked and passed the array of results

The key to all this is the closure which allows chainedRunner to keep track of completing functions. This is the key take away of each async method I will explore.

The code could easily be updated so that the results of each getData call would be passed to the next. Instead of binding values for each function, you could bind the first function and pass the result along with the wrapper callback to each subsequent callback. For simplicity's sake, I also neglected to handle error cases, but an extra value returned to the wrapper to indicate an error would be sufficient to handle errors.

## Using Async Callbacks in Parallel
If you think about the naive example, if each function was async, you would have parallel execution. But what if you wanted to do something only when all the functions have completed? As written, you can't.

{% highlight js %}
    // naive implementation
    for (let i = 0; i < max; i++) {
      arrayOfFunctions[i]();
    }
{% endhighlight %}

To do something similar but in parallel, you would want to call the final callback only after all the functions have completed. (Or reached an error, again, something I didn't implement for simplicity's sake.) Again we have a getData function that does an async action, and a getDataParallel function that binds the data before passing it to parallelRunner.

parallelRunner differs from chainedRunner in that it fires all functions it is passed without waiting for any of them to complete. Like, chainedRunner it passes the wrapper function to each function it calls. This will allow wrapper to keep track of function completion. Once all the functions have completed, it calls the final callback. To make it a little more useful, I added an option to keep the results ordered by the position in the function array of the function that generated the result. 

It should be noted, that if the functions are not async, all calls will happen in serially. 

{% highlight js %}
const getDataParallel = function (key, ordered, callback) {
    const funcs = [];

    for (let i = 0; i < key; i++) {
      funcs.push(this.getData.bind(undefined, i + 1));
    }

    parallelRunner(funcs, ordered, callback);
  }

const parallelRunner = (funcs, ordered, callback) => {
  //results store
  const results = [];
  //index of current iteration
  let countOfAddedItems = 0;

  //wrap the callback and add to the result array as the values are returned
  const wrapper = (index, value) => {

    if (index === null) {
      //unordered results
      results.push(value);
      countOfAddedItems++;
    } else {
      //ordered results
      results[index] = value;
      countOfAddedItems++;
    }

    if (countOfAddedItems === funcs.length) {
      callback(null, results);
    }
  };

  //call the functions
  for (let i = 0; i < funcs.length; i++) {
    funcs[i](wrapper.bind(null, ordered ? i : null));
  }

};

{% endhighlight %}

## Using Async Callbacks in Race
Finally, raceRunner is a method for calling a callback after the first function has completed. Keep in mind, that a functions that are fired in this method will run to completion. Using race with expensive resources might not be the best approach. 

{% highlight js %}
const raceRunner = (funcs, callback) => {
  let hasEnded = false;

  const wrapper = (value) => {

    //Only call the callback one time, 
    //otherwise let all others silently finish.
    if (!hasEnded) {
      callback(null, value);
      hasEnded = true;
    }
  }

  //call the functions
  for (let i = 0; i < funcs.length; i++) {
    funcs[i](wrapper);
  }
};
{% endhighlight %}

In my next blog posts, I will show how you can accomplish this with async.js, promises with bluebird, and with the async await syntax.