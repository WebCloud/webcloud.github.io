---
layout: post
title: "Memoizing with Javascript and closure - experiment"
date: 2014-05-07 19:59:07 +0200
comments: true
categories: [Javascript, basics]
---

## It's good to know the basics
On this quick blog post we will see a glimpse of the power of the Javascript's functional scope and the usage of closures applied to _memoization_. With the usage of closures you can have a well structured, modular and reusable code with it's use.

<!--more-->

{% blockquote @jeresig http://www.manning.com/resig/ Secrets of the JavaScript Ninja %}
Memoization (no, that’s not a typo) is the process of building a function that’s capable of remembering its previously computed values. This can markedly increase performance by avoiding needless complex computations that have already been performed.
{% endblockquote %}

This principle is very interesting and it's also addressed on Crockford's [JavaScript: The Good Parts](http://shop.oreilly.com/product/9780596517748.do), both are must have books for any Javascript programmer out there.

On Resig's book we have a demonstration of a memoizer to calculate prime numbers with the following code:

{% codeblock Memoizer Function lang:javascript %}
function isPrime(value) {
  if (!isPrime.answers) isPrime.answers = {};
  if (isPrime.answers[value] != null) {
    console.log('had in cache...');
    return isPrime.answers[value];
  }
  var prime = value != 1; // 1 can never be prime
  for (var i = 2; i < value; i++) {
    if (value % i == 0) {
      prime = false;
      break;
    }
  }
  console.log('hadn\'t in cache...');
  return isPrime.answers[value] = prime;
}
{% endcodeblock %}

# The closure way
Even though the code above is very good to demonstrate that functions are also first class objects, that might not be the most intended way to actually do that function. Mostly for security reasons, you don't want your internal objects used for control exposed to the global scope. Now, let's take a slightly different approach to the same problem.

{% codeblock Different approach on the memoizer Function lang:javascript %}
var isPrimeScoped = (function(){
  var answers = {};
  return function(value){
    if (answers[value] != null) {
      console.log('had in cache...');
      return answers[value];
    }
    var prime = value != 1; // 1 can never be prime
    for (var i = 2; i < value; i++) {
      if (value % i == 0) {
        prime = false;
        break;
      }
    }
    console.log('hadn\'t in cache...');
    return answers[value] = prime;
  };
}());
{% endcodeblock %}

The code above creates a auto executing anonymous function that will wrap the private variable `answers` into it's scope, after that it will return the function that will be stored on the `isPrimeScoped` variable to be used. Now, even though the first function gets executed and returned immediately, the variable `answers` will be available, only, for the returned function, making that variable it's private object.

Both functions are meant to be called the same way, and expected to have the same behaviour, but on the `isPrimeScoped` the 'cache' object `answers` is private and not accessible from the global scope as a property of that function.

{% codeblock Running both functions lang:javascript %}
isPrime(5) // logs "hadn't in cache..."
// returns true

isPrime(5) // logs "had in cache..."
// returns true

isPrime.answers // object {5: true}

isPrime.answers[5] = false // oops, I'm overwriting the control object

isPrime(5) // logs "had in cache..."
// returns false, wrongly

isPrimeScoped(5) // logs "hadn't in cache..."
// returns true

isPrimeScoped(5) // logs "had in cache..."
// returns true

isPrimeScoped.answers // undefined
{% endcodeblock %}

# Some further explanations
The availability of the `answers` object to the returned function happens because Javascript have a function scope, which means that whatever variable you write inside of a function is going to be accessible on the entire function body.

{% codeblock Different approach on the memoizer Function lang:javascript %}
function myFunction(){
  var first = "I can access this";

  console.log(second); // undefined

  if(true){
    var second = "And also this";
  }

  console.log(first); // "I can access this"
  console.log(second); // "And also this"
}
{% endcodeblock %}

For some developers comming from most, if not all, languages that might look strange. This is due to the Javascript compiler/runtimes, it picks any variable declaration and breaks into two parts. The first one is the declaration itself `var first = undefined, second = undefined;` and places on the very beginning of the function body. And the second part which is the value assignment and it's placed wherever the variable was first assigned. So the code above will be translated to someting similar to the following code:

{% codeblock Different approach on the memoizer Function lang:javascript %}
this.myFunction = function myFunction(){
  var first = undefined, second = undefined;

  first = "I can access this";

  console.log(second); // undefined

  if(true){
    second = "And also this";
  }

  console.log(first); // "I can access this"
  console.log(second); // "And also this"
}
{% endcodeblock %}

That is why the first log on the `second` variable is undefined, the variable was not asigned to any value up to that point, which will defaults to undefined.

You can also notice that the function declaration is modified to `this.myFunction = function myFunction(){...}`, this happens to make `myFunction` available to the global scope as window.myFunction, where 'this' refers to the window object on the global scope.

# Closure on the closure topic
Closures are very useful, and largely used when you want to protect objects from the global scope, if you browse through the jQuery code you will see it's use. On the global scope should be only objects/functions meant to be accessed by/from it.

Paul Irish's [10 Things I Learned from the jQuery Source video](https://www.youtube.com/watch?v=i_qE1iAmjFg) from 2010 (a, not so, long time ago) talks about that technique and more. you can also read about closure and other great techniques on the books mentioned on the beginning of this post.

_PS: I know that Resig don't say that's the best way to do it, I'm just making an experiment on the same subject_
