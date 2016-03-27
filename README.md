# Escape.js

_(Experiment; Work in Progress; Any kind of feedback is welcome!)_

Using escape analysis to enforce statically known heap value lifetime.

## TL;DR

Normally, heap values in JavaScript are Garbage Collected (GC). However, if
the compiler would have a way to figure out the lifetime of the value - it will
be able to deallocate it immediately, thus skipping the GC altogether.

The question is how to figure this out. This project handles this by doing
(and enforcing) a static escape analysis, disallowing values to be used after
they "escape". The place (function, object) where they escaped is responsible
for freeing them up.

(This concept is inspired by [borrowed pointers in rust][0]. No syntax sugar
was added to allow running this code in regular JS engines).

## Ways to escape


* `return value` - escapes current closure
* `outerVar = value` - assuming `outerVar` is in outer scope
* `outerVar.prop = value` - assuming `outerVar` is in outer scope
* `escaping.prop = value` - assuming `escaping` object escapes
* `escape(obj)` - assuming either of these two conditions:
  * the first argument of `escape` escapes it's  __closure__
  * the function signature is statically unknown (TODO(indutny): enforce this?)
* `value` is used in escaping closure (escaping statement is the `value`
  declaration)

If value does not escape - it is deallocated at the end of the scope where it
was declared. When value is deallocated - all its sub-values are deallocated too
(object property values, array values, function closure values).

## Disallowed things

* Use of escaped value in a statement reachable from the statement where it
  escaped.
* Circular references _(TODO(indutny): is this even possible? What other
  conditions do we need to enforce this statically?)_

## Examples

Some common setup:
```js
function escape(obj) {
  return obj;
}

function log(obj) {
  // ...obj does not escape here...
}
```

```js
function ex1() {
  let value = 'some string';

  // no escape here, `log` is known and its argument does not escape
  log(value);

  // Value can be copied, but copies are known statically and used in escape
  // analysis
  let copy = value;
  let copy2 = value;

  // `value` is deallocated here, `copy`/`copy2` are not deallocated
}
```

```js
function ex2() {
  let value = 'some string';

  {
    // `value` escapes here
    let r = escape(value);

    // `r` holds the same pointer as `value`
    // `r` is deallocated at the end of this scope
  }

  // `value` is no longer usable here, because we passed it to `escape`
  // This statement will throw during the compilation
  log(value);

  // `value` is **not** deallocated here
}
```

```js
function ex3() {
  let one = {};
  let two = {};

  one.two = two;

  // Circular references are disallowed, compiler will error on this statement
  two.one = one;
}
```

```js
function ex4() {
  let value = 'some string';

  // `closure` does **not** "capture" `value`, because `closure` itself does not
  // escape
  function closure() {
    return value;
  }

  // Static analysis is used to merge `x`, `y`, and `value`
  let x = closure();
  let y = closure();

  // `value` can be used here
  log(value);

  // `x`, `y`, `value` escape
  escape(y);

  // This will throw, because `value` escaped
  log(x);

  // `value` is **not** deallocated here
}
```

```js
function ex5() {
  let value = 'some string';

  // `closure` **captures** `value`, but only after `escape(closure)`
  function closure() {
    return value;
  }

  // This statement will throw during compilation, `closure` escapes so we
  // can't create copies of `value`
  let copy = closure();

  escape(closure);

  // `closure` escaped, can't be called
  // This statement will throw during compilation
  closure();

  // `value` is **not** deallocated here
}
```

## LICENSE

This software is licensed under the MIT License.

Copyright Fedor Indutny, 2016.

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to permit
persons to whom the Software is furnished to do so, subject to the
following conditions:

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN
NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR
OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE
USE OR OTHER DEALINGS IN THE SOFTWARE.

[0]: http://doc.rust-lang.org/book/references-and-borrowing.html#borrowing
