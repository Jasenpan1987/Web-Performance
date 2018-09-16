# 1. Cost of Javascrip

## 1.1 Parsing JS

Parsing javascript is one of the most expensive stuff in all javascript actions. For most of time, when we see javascript is working on the browser, it is actually parsing and compiling, not actually running.

Javascript uses JIT (just in time) compilation, it compiles just before its execution. And each of the browser engine has good and bad parts when parsing and compiling javascript.

## 1.2 Javascript and browser engine workflow

1. brwoser download the javascript from cloud or cdn, and for now, it treat the code like string.
2. browser starts to parse it, and the javascript parser will create a AST (abstract syntax tree) of the code.
3. the AST will be sent to the javascript interpreter, and the interpreter will eventually generate the binary code for our computer to consume, but before that...
4. some part of the code will be sent to another compiler called Optimizing compiler or the "turbo fan" in v8, to generate optimized binary code.

**In general, our goal of optimizing javascript is try to make as much of our code to be compilable for the "turbo fan" as possible.**

### 1.2.1 Parsing

Parsing is slow, and even slower on mobile devices. And parsing happens in two phases:

- Eager parsing (full parsing): this is what you think of when you think about parsing
- Lazy parsing (pre-parsing): do bear minimum now, we will parse it later.

One quick example

```js
function sumOfSqr(x, y) {
  function sqr(a) {
    return a * a;
  }

  return sqr(x) + sqr(y);
}
```

is more costy than the following when parsing

```js
function sumOfSqr(x, y) {
  return sqr(x) + sqr(y);
}

function sqr(a) {
  return a * a;
}
```

### 1.2.2 The optimizing compiler

How does the optimizing compiler works?

```js
let iterations = 1e7;

const a = 1;
const b = 2;

const add = (x, y) => x + y;

while (iterations--) {
  add(a, b);
}
```

In the above code, since the `add` function (the "+" sign) always add two numbers togather, it will become optimized. But... what if we do this,

```js
add("foo", "bar");
```

The add function will be deoptimized, because the "adding two number" rule is no longer true anymore, so `add` function will be deoptimized.

The optimizing compiler optimizes for what it's seen. If it sees something new, that's problematic.

**Rule 1: calling a function with different types of arguments are bad performance wise.**

### 1.2.3 \*-morphism

- **Monomorphic**: This is all I know and all I have seen, and I can get incredibly faster at this one thing.
- **Polymorphic**: I have seen a few shapes before, let me just check to see which one it is and I will do the faster thing.
- **Megamorphic**: I've see things, a lot of things, I don't know which one it is, sorry.

Each browser has a special hidden function to evaluate the type of two objects are the same one, in v8, it's `%HaveSameMap`

```js
// this code only works on node with --allow-native-syntax flag on
var a = { x: 1 },
  b = { y: 2 },
  c = {x:5},
  d={x:1000000000000}
  e={x: 2, y:3};

%HaveSameMap(a, b); // false
%HaveSameMap(a, c); // true
%HaveSameMap(a, d); // false
%HaveSameMap(a, e); // false
```

## 1.3 Hidden class, scope and prototyping

Check this code

```js
function test() {
  class Point {
    constructor(x, y) {
      this.x = x;
      this.y = y;
    }
  }

  function add(point) {
    return point.x + point.y;
  }

  const point = new Point(10, 20);
  add(point);
}
// run testing
let iterations = 1000000;
while (iterations--) {
  test();
}
```

This code will take about 0.8 second to run. But... what if we do this

```js
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}

function test() {
  function add(point) {
    return point.x + point.y;
  }

  const point = new Point(10, 20);
  add(point);
}
```

This code will only take about 9 milliseconds to run, and I think move the class defination is not the only reason to gain such a huge performance boost.

Let's try this

```js
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}

var a = new Point(10, 20);
var b = new Point(10, 20);

%HaveSameMap(a, b)
```

OK, this gives me a true, but what about this

```js
function makePoint() {
  class Point {
    constructor(x, y) {
      this.x = x;
      this.y = y;
    }
  }
  return new Point(10, 20)
}

var a = new makePoint();
var b = new makePoint();

%HaveSameMap(a, b)
```

**FALSE**, this is because the prototype chain changed. So, this explains why the above two implementation has so much difference in performance.

## Takeaways:

1. Initialize properties at creation
2. Initialize them in the same order
3. Try not modify them after creation
4. Maybe use TS or flow so we don't have to worry about these things.

## 1.4 Function Inlining

```js
function sqr(x) {
  return x * x;
}

function sumSqr1(a, b) {
  return sqr(a) + sqr(b);
}

function sumSqr2(a, b) {
  return a * a + b * b;
}
```

For the above two implementation of `sumSqr`, we expect the second one will have much more better performance than the first one, because it doesn't have to go to the implementation of `sqr` again and again. BUT, they are about the same in performance wise, Why? because javascript engine has the **inlining** functionality, which means, if you need to call a function again and again, it will grab the implementation of that function and rewrite the current code.

## 1.5 Takeaways

1. The easiest way to reduce the parse, compile, and execution time is to shipe less code.
2. Use the **User Timing API** to figure out where is the biggest amount of hurt is.
3. Consider to use a type system, so that we don't have to worry about the hidden typing system in the javascript engine.
