+++
author = "Saharsh Singh"
title = "Arithmetic Curry, Coming Right Up"
slug= "arithmetic-curry"
date = "2019-11-18"
description = "In this article I reinvent arithmetic from scratch (kind of) while showcasing some cool programming techniques – currying, partial application, and recursion."
tags = [
    "code",
    "functional programming",
    "math",
    "javascript",
    "software",
    "tech",
    "work",
]
categories = [
    "articles",
]
aliases = [
    "/2019/11/18/arithmetic-curry-coming-right-up/",
]
+++

In this article I reinvent arithmetic from scratch (kind of) while showcasing some cool programming techniques – currying, partial application, and recursion.

<!--more-->

One of my favorite blog articles is **Scott Aaronson**’s famous [Who Can Name the Bigger Number?](https://www.scottaaronson.com/writings/bignumbers.html). I highly recommend the article. It’s a very interesting perspective on human progress and the role big (well, really really big) numbers play in it. The relevant part of that article that I will discuss here is the Wilhelm Ackermann’s idea of an endless procession of arithmetic operations. I will quote Aaronson here to describe this sequence as he does a better job of it than I probably can.

> And so we arrive in the early twentieth century, when a school of mathematicians called the formalists sought to place all of mathematics on a rigorous axiomatic basis. A key question for the formalists was what the word ‘computable’ means. That is, how do we tell whether a sequence of numbers can be listed by a definite, mechanical procedure? Some mathematicians thought that ‘computable’ coincided with a technical notion called ‘primitive recursive.’ But in 1928 Wilhelm Ackermann disproved them by constructing a sequence of numbers that’s clearly computable, yet grows too quickly to be primitive recursive.

> Ackermann’s idea was to create an endless procession of arithmetic operations, each more powerful than the last. First comes addition. Second comes multiplication, which we can think of as repeated addition: for example, 5´3 means 5 added to itself 3 times, or 5+5+5 = 15. Third comes exponentiation, which we can think of as repeated multiplication. Fourth comes … what? Well, we have to invent a weird new operation, for repeated exponentiation. The mathematician Rudy Rucker calls it ‘tetration.’ For example, ‘5 tetrated to the 3’ means 5 raised to its own power 3 times, or , a number with 2,185 digits. We can go on. Fifth comes repeated tetration: shall we call it ‘pentation’? Sixth comes repeated pentation: ‘hexation’? The operations continue infinitely, with each one standing on its predecessor to peer even higher into the firmament of big numbers.

I won’t lie. It has been a while since I have examined arithmetic this closely. I just take addition, multiplication, and even exponentiation for granted at this point. However, thinking of them as the initial few elements in an infinite sequence of operations was a bit mind blowing. Mathematicians call this sequence of operations [the hyperoperation sequence](https://en.wikipedia.org/wiki/Hyperoperation) and use the [Knuth up-arrow notation](https://en.wikipedia.org/wiki/Knuth%27s_up-arrow_notation) to symbolize them. At this point, my regular readers, assuming they exist, are saying “But you usually only write about coding stuff Saharsh! What’s with all the math talk?” Don’t worry, this is still very much a software development blog. Armed with this perspective of arithmetic, I want to showcase the functional programming technique of currying, partial application, and recursion.

[Currying](https://en.wikipedia.org/wiki/Currying) is a useful technique used to decompose functions that take multiple arguments into a series of functions that each take only one argument. Currying is a powerful concept that programmers use to create [partially applied functions](https://en.wikipedia.org/wiki/Partial_application) from higher order functions, allowing for flexible and elegant code. That’s a lot to unwrap if you haven’t seen this before. So let’s write some real code and use currying to implement the hyperoperation sequence. In the process, we will create arithmetic out of thin air in this article (kind of). I’ve published all the code relevant to this article as [a Github gist](https://gist.github.com/saharshsingh/7c7cfcd7427f5f62c2613702c0cfe13b). You can copy the entirety of the gist into your favorite browser’s developer console and start testing commands like `add(2)(3)`, `multiply(5)(6)`, and `power(2)(3)`.

Before proceeding further, I would like to issue a disclaimer. The code in the gist above is for educational and intellectual curiosity purposes only. There’s a reason no one actually implements their Math libraries this way. First of all, the operations in the gist are really only valid as long as inputs are positive integers. Second, and more importantly, since I ultimately implement every arithmetic operation by painfully and tediously adding one to some number over and over again, it’s needless to say these implementations are ridiculously inefficient. Alright, with that in mind let’s break this gist down.

So, the hyperoperation sequence actually starts with [the successor function](https://en.wikipedia.org/wiki/Successor_function) (programmers typically refer to it as incrementing). In the gist, I implement the `increment` function using Javascript’s built-in pre-increment operator. I also go ahead and implement a `decrement` function which will come in handy in implementing the other elements of the hyperoperation sequence. Great! So the first element of the sequence is easy.

{{< highlight javascript "linenos=table" >}}
// A little cheat to move to right on the integer line
const increment = i => ++i

// A little cheat to move to the left on the integer line
const decrement = i => --i
{{< /highlight >}}

Next up is addition. Addition, second element in the hyperoperation sequence, is repeated incrementing. For example, `2+3` is incrementing `2` three times.

* `2 + 1 = 3`
* `3 + 1 = 4`
* `4 + 1 = 5`

Therefore, before I implement addition, first I will implement the concept of iterating as it applies to the hyperoperation sequence. If you notice, iteration here involves the following steps.

1. Accept a function, an initial arugment, and number of times the function needs to be executed
1. Compute a result by executing the function by feeding it the initial argument
1. Repeat the second step as many as times as told in first step, replacing the argument for each subsequent execution with the result of the previous execution.

Implementing this using recursion is straight forward.

{{< highlight javascript "linenos=table" >}}
const iterate = (func, arg, executionCount) => {
    if(executionCount === 0) {
        return arg
    }
    return iterate( func, func(arg), decrement(executionCount) )
}
{{< /highlight >}}

The `iterate` function above is [tail-recursive](https://en.wikipedia.org/wiki/Tail_call) and would work just fine in a “real” functional programming language like Haskell. However, javascript engines that don’t do [tail call optimization](https://stackoverflow.com/a/54721813) (which is pretty much all of them except Safari) will run into stack overflow errors. Following implementation side steps this by using iteration instead of recursion to implement the same logic. There is a reason I issued the disclaimer above.

{{< highlight javascript "linenos=table" >}}
const iterate = (func, arg, executionCount) => { 
    let result = arg;
    for (let i = 0; i < executionCount; i = increment(i)) {
        result = func(result);
    }
    return result;
}
{{< /highlight >}}

With the iterate function in place, defining addition becomes straightforward.

{{< highlight javascript "linenos=table" >}}
// Curried declaration of addition of natural numbers implemented
// as an iteration of increment
const add = a => b => iterate (increment, a, b)
{{< /highlight >}}

Notice how the implementation just states what addition is according to the hyperoperation sequence. When we add two numbers, we increment the first number as many times as the second number. Once again, I must issue my previous disclaimer. This is not the fully-featured addition you are used to. This function only really works reliably with natural numbers, or in other words positive integers. The second argument is used as the value that specifies how many iterations of the function to execute. So what does `-5 times` mean? In our `iterate` implementation, negative numbers might as well be `0` as you will just get back the first argument as the result.

Instead the point here is to implement addition in code the way we first learn to do it in grade school. Also, the point is to see currying in action. Here we don’t see benefits of currying just yet, but we do see how to declare a curried function in javascript (technically [ECMAScript](https://en.wikipedia.org/wiki/ECMAScript#6th_Edition_-_ECMAScript_2015)). We use javascript’s arrow operator to declare add as a curried function. First we have a function that takes one argument, a,  and returns another function that also takes one argument, b. This second function will return an integer which will be result of adding a and b. How does declaring the add function like this benefit us? Let’s find out by implementing multiply.

Multiplication, the next element, is repeated addition.

* `2 X 3 = 2 + 2 + 2 = 6`

So, when you multiply `2` and `3`, you are adding two to itself twice (or one less than three times). Given our `decrement`, `iterate`, and `add` functions, we can implement `multiply` by simply stating this idea in javascript.

{{< highlight javascript "linenos=table" >}}
// Curried declaration of multiplication of natural numbers implemented as an
// iteration of add
const multiply = a => b => iterate ( add(a), a, decrement(b) )
{{< /highlight >}}

Notice the first argument to `iterate` here is `add(a)`. The `iterate` function takes a “single argument function” as its first argument. So we give it just that by partially applying our curried `add` function and composing a function that is equivalent to `a+`. The second argument to the `iterate` function is `a`. As a result, on the first iteration, the result will be `a+a`. On the second iteration, we will feed a+a to the a+ function and get `a+(a+a)` as a result. Similarly, we will get `a+(a+(a+a))` as the result of the third iteration after applying the `a+(a+a)` input to the `a+` function. We will iterate like this `b-1` times, the result of the third argument to the `iterate` function. Let’s see how this looks for `multiply(3)(5)`.

* `multiply(3)(5)` = `iterate( add(3), 3, decrement(5) )` = `iterate( 3+, 3, 4)`
* Iteration 1 : `3 + 3 = 6`
* Iteration 2 : `3 + 6 = 9`
* Iteration 3 : `3 + 9 = 12`
* Iteration 4 : `3 + 12 = 15`
* Final result : `15`

Keep in mind each time we are executing the `3+` function, we are executing the `add` function from above where we increment `3` as many times as the value to the right. So when we do `3+12`, that results in the increment function being called twelve times. Next in the hyperoperation sequence is exponentiation, also known as the power function.

{{< highlight javascript "linenos=table" >}}
// Curried definition of power operation using natural numbers implemented as an
// iteration of multiply
const power = a => b => iterate (multiply(a), a, decrement(b) )
{{< /highlight >}}

The idea here is exactly the same as `multiply`, except we are now iterating the `multiply` operation instead of the `add` operation. The `power` function grows fast. If you stick with the recursive implementation of `iterate`, you will start getting stack overflow errors as quickly as `power(2)(14)`, `power(4)(8)`, and `power(25)(3)`. Using the iteration-based implementation will get you much further, but you will notice computation times getting noticeably long fairly quickly. However, exponential growth is still nothing compared to what comes next in the hyperoperation sequence. Instead of defining each subsequent element in the sequence separately, mathematicians generalize the sequence using the [Knuth’s up-arrow notation](https://en.wikipedia.org/wiki/Knuth%27s_up-arrow_notation#Definition).

Starting from exponentiation, each subsequent operation in the hyperoperation sequence is denoted using additional arrows. So, `power(2)(3)` is `2↑3`. The operation that iterates exponentation, tetration, is denoted using two arrows. So `tetration(2)(3)` is `2↑↑3`. Following recursive formula is used to calculate values for the operations described using the up-arrow notation.

{{< svg id="posts/arithmetic-curry-coming-right-up/up-arrow-def" >}}

Using recursion, currying (or rather partial application of a curried function), and other operations in our gist, we can easily implement this formula in javascript.

{{< highlight javascript "linenos=table" >}}
// Curried definition of Knuth's up-arrow notation
// See https://en.wikipedia.org/wiki/Knuth%27s_up-arrow_notation
const arrow = a => arrowNum => b => {

    // base case is just the power operation
    if (arrowNum === 1) {
        return power(a)(b);
    }

    // general case is a recursive operation
    const arrowMinusOne = decrement(arrowNum)
    const bMinusOne = decrement(b)
    return iterate ( arrow (a) (arrowMinusOne), a, bMinusOne );
}
{{< /highlight >}}

The `arrow` function takes three arguments. Like `add`, `multiply`, and `power`, there are the usual suspects, `a` and `b`. However, we also add `arrowNum` in the middle to express how many arrows are being used. The recursive implementation uses `arrowNum === 1` as the base case where the `arrow` function becomes the `power` function. For cases where `arrowNum` is greater than one, the operation iterates a lower operation where the number of arrows is one less than `arrowNum`. The pattern here is identical to what we did for `multiply` and `power`. You will notice the `arrow` function is a curried function, and the `arrow` function partially applies itself when calling `iterate`. Thus the recursion happens partially within the `arrow` function and then fully further down the call stack in the `iterate` function.

Let’s make more sense of this by resolving example instances of the arrow operation. But first let’s examine `multiply` and `power` one more time.

* `3 X 5` = `multiply(3)(5)` = `iterate ( add(3), 3, 4 )`
* `4 ^ 6` = `power(4)(6)` = `iterate ( multiply(4), 4, 5 )`

We can resolve the `arrow` function similarly. Let’s look at two different instances: `2↑↑3`, and `2↑↑↑4`. The first one has fewer layers to peel.

* `2 ↑↑ 3` = `arrow (2) (2) (3)` = `iterate ( arrow(2)(1), 2,  2 )`
* The `arrow(2)(1)` function, or `2↑`, is a partially applied power function, `2^`.
* Iteration 1: `2 ↑ 2` = `2 ^ 2` = `4`
* Iteration 2: `2 ↑ 4` = `2 ^ 4` = `16`
* Final Result: `16`

The second expression, `2↑↑↑4`, expands a bit more.

* `2 ↑↑↑ 4` = `arrow (2) (3) (4)` = `iterate ( arrow(2)(2), 2, 3 )`
* Iteration 1: `2 ↑↑ 2` = `iterate ( 2↑, 2, 1 )` = `2 ^ 2` = `4`
* Iteration 2: `2 ↑↑ 4` = `iterate ( 2↑, 2, 3)`
* Iteration 2, 1: `2 ↑ 2` = `2 ^ 2` = `4`
* Iteration 2, 2: `2 ↑ 4` = `2 ^ 4` = `16`
* Iteration 2, 3: `2 ↑ 16` = `2 ^ 16` = `65536`
* Iteration 3: `2 ↑↑ 65536` = `iterate ( 2↑, 2, 65536)`
* Final Result: `2 ^ 2 … ^ 2` (where you have `65536` copies of `2`)

Operations in the hyperoperation sequence beyond exponentiation quickly get out of hand. There is no computer on this planet that can give you all the digits to the number computed by the last expression. A tower of exponents made of `2`s stacked on top of themselves `65536` times is a ridiculously big number. Much, much, much bigger than the number of all the atoms in the visible universe. So obviously our browsers are not going to be much help in getting an answer, especially considering my grossly inefficient implementation. However, there are some expressions in tetration and pentation that have [reasonably computable answers](https://en.wikipedia.org/wiki/Knuth%27s_up-arrow_notation#Tables_of_values). Anyways, as stated above, that’s not really the point here anyways. Instead, I hope you were able to get better appreciation for currying, partial application, and recursion. Also, check out the Scott Aaronson blog I mentioned in the beginning of this article. It’s pretty cool.
