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

# 2. Rendering performance

## 2.1 DOM and CSSOM

When rendering a page, the browser will go through the html page to create a DOM, and also create a CSSOM in order to deal with the css stuff, such as classes, properties, selectors, animations... So here are some rules to make this generating CSSOM process faster:

1. Reduce the amount of unused css code, the less styles we have, the faster for the browser to check

2. Reduce the number of styles that effects on an element
3. When possible, try to use class names instead of nested or complex selectors
4. Try use naming convensions such as BEM

## 2.2 Javascript and render pipeline

Javascript can manipulate the DOM on a page, it can adding and removing the class of an element, it can add styles onto an element, it can add new elements or delete exsisting elements...

There is a pipeline for the browser to do such things:

**javascript -> style/class -> layout -> paint -> composite**

All these steps happening in one single thread on computer and it will result in poor performance. But, we can acutally skip some of these processes.

### 2.2.1 layout and reflows

When ever the geometry of an element changes, the browser has to reflow the page.
This is the second most expansive step in the entire workflow, and it's an block operation, and it will be noticeable by the user if it happens too often.

Reflow of an element causes the relow of it's parents and children in the tree.

And the most expensive thing is repaint, which caused by reflow of layout.

**How to Avoid reflow**

1. change classes at lowest level of the DOM tree.
2. avoid repeatly modifing inline styles.
3. Trade some smoothness for speed if you are doing an animation in javascript
4. avoid table layout
5. batch DOM manipulations
6. debounce window resizing events

exercise 1:

https://codepen.io/stevekinney/full/eVadLB/

Let's say we have a button and a bunch of divs, and when the button get clicked, all the divs will double their sizes:

```html
<style>
#boxes {
  margin: 1em 0;
}

.box {
  background-color: black;
  height: 20px;
}
</style>
<button id="double-sizes">Double Sizes</button>
<section id="boxes">
  <div class="box" style="width: 20px;"></div>
  <div class="box" style="width: 5px;"></div>
  <div class="box" style="width: 16px;"></div>
  <div class="box" style="width: 5px;"></div>
  ....... a lot of more
</section>
```

Version 1

```js
const button = document.getElementById("double-sizes");
const boxes = Array.from(document.querySelectorAll(".box"));

const doubleWidth = elem => {
  const width = elem.offsetWidth;
  elem.style.width = `${width * 2}px`;
};

button.addEventListener("click", event => {
  boxes.forEach(doubleWidth);
});
```

The above code has very bad performance and we can verify it from the chrome developer tool (the performance section, record). Now let's try to improve it

```js
...

button.addEventListener("click", event => {
  const widths = boxes.map(box => box.offsetWidth);

  boxes.forEach((box, idx) => {
    box.style.width = `${widths[idx] * 2}px`
  })
})
```

If you check it again, it's much more better. So what did we just do?

### 2.2.2 Layout thrashing / forced sychoronous layout

When we read the layout, for example getting its width and height, and violently write its layout multiple times causing document reflows.

```js
firstElement.classList.toggle("bigger"); // change
const firstElemWidth = firstElement.width; // calculate
secondElement.classList.toggle("bigger"); // change
const secondElemWidth = secondElement.width; // calculate
```

This is bad, and this is the first thing we look for, the browser knew it was going to have to change the stuff after the first line. Then you went ahead and ask for it for some information about the geometry of another object, so the brwoser stopped your javascript and reflowed the page in order to get your an answer.

And this is why we want to banch all DOM manipulation togather like the following:

```js
firstElement.classList.toggle("bigger"); // change
secondElement.classList.toggle("bigger"); // change
const firstElemWidth = firstElement.width; // calculate
const secondElemWidth = secondElement.width; // calculate
```

exercise 2:
/exercise npm start -> layout fun

`/moving-boxes/script.js`

```js
const elements = Array.from(document.querySelectorAll(".element"));
// bad
registerNextClick(function(timestamp) {
  elements.forEach((element, index) => {
    const top = element.offsetTop;
    const nextPosition = +(
      ((Math.sin(top + timestamp / 1000) + 1) / 2) *
      containerWidth
    );
    element.style.transform = `translateX(${nextPosition}px)`;
  });
});
```

This is a very bad code snipet, it has the same thrashing problem, it trying to calculate the style values and set it in the same run. We can use the last trick of **batching operations** to improve it

```js
// better
registerNextClick(function(timestamp) {
  const nextPositions = elements.map(elem => {
    return +(
      ((Math.sin(elem.offsetTop + timestamp / 1000) + 1) / 2) *
      containerWidth
    );
  });
  elements.forEach((element, index) => {
    element.style.transform = `translateX(${nextPositions[index]}px)`;
  });
});
```

Another approach to reduce layout thrashing is by using a `requestAnimationFrame`, this function is a window function, it basically tells the window, I don't what the stuff happening now, do it later during your next repaint (https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame)

```js
registerNextClick(function(timestamp) {
  elements.forEach((element, index) => {
    const top = element.offsetTop;
    const nextPosition = +(
      ((Math.sin(top + timestamp / 1000) + 1) / 2) *
      containerWidth
    );
    requestAnimationFrame(() => {
      element.style.transform = `translateX(${nextPosition}px)`;
    });
  });
});
```

What if we combine them togather?

```js
registerNextClick(function(timestamp) {
  const nextPositions = elements.map(elem => {
    return +(
      ((Math.sin(elem.offsetTop + timestamp / 1000) + 1) / 2) *
      containerWidth
    );
  });
  elements.forEach((element, index) => {
    requestAnimationFrame(() => {
      element.style.transform = `translateX(${nextPositions[index]}px)`;
    });
  });
});
```

The `requestAnimationFrame` solution also has some issues, one issue is it fires so many times and we don't actually need that many times.

## 2.3 FastDom

fastdom is a library on github (https://github.com/wilsonpage/fastdom), it's a pretty small library, and what it does is batching the DOM read and write in togather and use `requestAnimationFrame` to do them once.

```js
registerNextClick(function(timestamp) {
  elements.forEach((elem, idx) => {
    fastdom.measure(() => {
      var top = elem.offsetTop;
      const nextPosition =
        ((Math.sin(top + timestamp / 1000) + 1) / 2) * containerWidth;

      fastdom.mutate(() => {
        elem.style.transform = `translateX(${nextPosition}px)`;
      });
    });
  });
});
```

### 2.3.1 Framework

Do the frameworks handle the layout thrashing issue? Yes, they do, most of the frameworks such as React, Angular and Ember have solved the layout thrashing out of the box, so that we don't have to worry about it anymore. For React, we MUST use the production mode, huge difference.

### 2.3.2 Some Takeaways

1. Don't mix reading and writing layout properties (width, height, color...) togather, you are doing unnecessary works (thrashing).
2. If you can change the visual appreances of an element by adding css class, do that approach, you are avoid accidental thrashing.
3. Storing data in memory (store all the heights in an array...) - as opposed to DOM - means we don't have to check the DOM.
4. Frameworks are generally good but with some overhead.
5. You don't need a framework to take advantage of this.
6. You can do bad things even with a framework.
7. You may not always know wheather you are layout thrashing or not, so measure it use the developer tool.

## 2.4 Painting

Anything you change something other than opacity or css transform... you are triggering a repaint.

Every layout change will cause a repaint, but not all the repaint need layout change (change color).

General speaking, when we have page in a browser, three thread will fire up:

1. UI thread, means the chrome tool bar, address bar, scroll bar... we can't do much about this thread.

2. The renderer thread, or the main thread, this is where all the javascript, parsing HTML and css, styles, calculations, layout, painting happens, there are one of these per thread.

3. The Compositor thread, which draw bitmaps to the screen via GPU.

Our optimization rule is: move the stuff from the renderer to the compositor thread, hand over stuff from CPU to GPU. And how do we achieve this? We manage layers.

**_Disclaimer: Compositing is kind of a hack, the w3c has not officially support it_**

You can't control layers, because it's an optimization that the browser does it for you under the hood. But you can influcen the browser to create the layers for you.

### 2.4.1 What kind of things has its own layer?

- The root object of the page.
- Objects that have specific css positions.
- Objects with css transform.
- Objects that have overflow.

### 2.4.2 the **will-change** property

You can give the browser hints by using the "will-change" property.

```css
.side-bar {
  will-change: transform;
}
```

It won't work on IE and edge, but won't break them either.

This code basically "suggest" the browser that the `side-bar` should be on its own layer. And we can force it by doing this:

```css
.side-bar {
  will-change: transformZ(0);
}
```

This is a hack of the hacks, don't use it!!!

#### Layer rules:

1. Managing Layers also has a cost for the browser, so NEVER do this

```css
* {
  will-change: transform;
}
```

The reason is it make everything its own layer, it will cost way more than the repainting work.

2. The Retina devices is come with layers for free, meaning you don't have to put the `will-change` property to suggest the layer to the browser.

### An layer promoting example

A smart way of promoting layers is tell the browser that the element will change shortly, for example, when we have an `<a>` tag, when the user hovers it, it is very likely that the user will click on it in the next second. So we can hint the browser that the `<a>` tag might need to be its own layer before it actually getting clicked.

```css
.sidebar {
  will-change: transform;
  transition: transform 0.5s;
}

.sidebar:hover {
  will-change: transform;
}

.sidebar.open {
  transform: tranlate(400px);
}
```

We can do it in javascript, and it is more likely to happen in the javascript

```js
myLink.addEventListener("mouseenter", () => {
  myLink.style.willChange = "transform";
});
```
