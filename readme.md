# Smart pipelines
ECMAScript Stage-(−1) Proposal by J. S. Choi, 2018-02.

This repository contains the formal specification for a proposed “smart pipe operator” `|>` in JavaScript. It is currently not even in Stage 0 of the [TC39 process][TC39 process] but it may eventually be presented to TC39.

## Background
The binary operator proposed here would provide a backwards- and forwards-compatible style of chaining nested expressions into a readable, left-to-right manner. Nested transformations become untangled into short steps in a zero-cost abstraction. The operator is similar to operators from numerous other languages, variously called “pipeline”, “threading”, and “feed” operators:

* [Clojure’s `->` and `as->`][Clojure pipe]
* [Forth’s, Joy’s, Factor’s, Onyx’s, PostScript’s, and RPL’s term concatenation][concatenative programming]
* [Elixir/Erlang’s `|>`][Elixir pipe]
* [Elm’s `|>`][Elm pipe]
* [F#’s `|>`][F# pipe]
* [Hack’s `|>` and `$$`][Hack pipe]
* [Julia’s `|>`][Julia pipe]
* [LiveScript’s `|>`][LiveScript pipe]
* [OCaml’s `|>`][OCaml pipe]
* [Perl 6’s `==>`][Perl 6 pipe]
* [R / magrittr’s `%>%`][R pipe]
* [Unix shells’ and PowerShell’s `|` ][Unix pipe]

It is also conceptually similar to [WHATWG-stream piping][] and [Node-stream piping][].

The proposal is a variant of a [previous pipe-operator proposal][] championed by [Daniel “littledan” Ehrenberg of Igalia][]. This variant is listed as [Proposal 4: Smart Mix on the pipe-proposal wiki][]. The variant resulted from [previous discussions about pipeline placeholders in the previous pipe-operator proposal][previous pipeline-placeholder discussions], which culminated in an [invitation by Ehrenberg to try writing a specification draft][littledan invitation]. A prototype Babel plugin is also brewing.

You can take part in the discussions on the [GitHub issue tracker][]. When you file an issue, please note in it that you are talking specifically about “Proposal 4: Smart Mix”.

This specification uses `#` as its topic token, but note that this is not set in stone. In particular, `@` could also be used. would be similarly terse, visually distinguishable, and easily typeable. [Bikeshedding over what characters to use for the topic token is occurring in a GitHub issue](https://github.com/tc39/proposal-pipeline-operator/issues/91).

## Motivation
Here is a table of code blocks in current JavaScript. Several are taken from several real-world code libraries.

<table>

<thead>

<tr>
<th>Source
<th>Status quo
<th>With smart pipes
<th>Notes

</thead>

<tbody>

<tr>
<td>

**[Prior pipeline proposal][]**.<br>
[Gilbert “mindeavor”][mindeavor] &c.<br>
ECMA International.<br>
2017–2018.<br>
BSD License.

<td>

```js
function doubleSay (str, separatorStr) {
  return `${str}${separatorStr}${str}`
}

function capitalize (str) {
  return str[0].toUpperCase()
    + str.substring(1)
}

new User.Message(
  capitalizedString(
    doubledSay(
      (await stringPromise)
        ?? throw new TypeError()
    )
  ) + '!'
)
```

<td>

```js
function doubleSay (str, separatorStr) {
  return `${str}${separatorStr}${string}`
}

function capitalize (str) {
  return str[0].toUpperCase()
    + str.substring(1)
}

stringPromise
  |> await #
  |> # ?? throw new TypeError()
  |> doubleSay(#, ', ')
  |> capitalize |> # + '!'
  |> new User.Message
```

<td>

Note that `|> capitalize` is a tacit unary function call; `|> capitalize(#)` would work but the `#` is unnecessary.<br>
Ditto for `|> new User.Message`, which is a tacit unary constructor call, abbreviated from `|> new User.Message(#)`.

<tr>
<td>″″
<td>″″
<td>

```js
stringPromise
  |> await #
  |> # ?? throw new TypeError()
  |> `${#}, ${#}`
  |> #[0].toUpperCase() + #.substring(1)
  |> # + '!'
  |> new User.Message
```

<td>When tiny functions are only used once, and their bodies would be self-documenting, they are ritual boilerplate that a programmer’s style may prefer to inline.

<tr>
<td>

**[Underscore.js][]** 1.8.3.<br>
[Jeremy Ashkenas][jashkenas] &c.<br>
2009–2018.<br>
MIT License.

<td>

```js
function (obj, pred, context) {
  var key;
  if (isArrayLike(obj)) {
    key = _.findIndex(obj, pred, context);
  } else {
    key = _.findKey(obj, pred, context);
  }
  if (key !== void 0 && key !== -1)
    return obj[key];
}
```

<td>

```js
function (obj, pred, context) {
  return obj
    |> isArrayLike(#) ? _.findIndex : _.findKey
    |> #(obj, pred, context)
    |> (# !== void 0 && # !== -1)
      ? obj[#] : undefined;
}
```

<td>

<tr>
<td>″″
<td>

```js
function (obj, pred, context) {
  return _.filter(obj,
    _.negate(cb(pred)),
    context
  )
}
```

<td>

```js
function (obj, pred, context) {
  return pred
    |> cb
    |> _.negate
    |> _.filter(obj, #, context)
}
```

<td>

<tr>
<td>″″
<td>

```js
function (
  srcFn, boundFn,
  ctxt, callingCtxt, args
) {
  if (!(callingCtxt instanceof boundFn))
    return srcFn.apply(ctxt, args);
  var self = baseCreate(srcFn.prototype);
  var result = srcFn.apply(self, args);
  if (_.isObject(result)) return result;
  return self
}
```

<td>

```js
function (
  srcFn, boundFn, ctxt, callingCtxt, args
) {
  if (callingCtxt |> # instanceof boundFn |> !#)
    return srcFn.apply(ctxt, args);
  var self = srcFn
    |> #.prototype |> baseCreate;
  return self
    |> srcFn.apply(#, args)
    |> _.isObject(#) ? # : self;
}
```

<td>

<tr>
<td>″″
<td>

```js
function (obj) {
  if (obj == null) return 0;
  return isArrayLike(obj)
    ? obj.length
    : _.keys(obj).length;
}
```

<td>

```js
function (obj) {
  if (obj == null) return 0;
  return obj |> isArrayLike
    ? obj.length
    : obj |> _.keys |> #.length;
}
```

<td>

<tr>
<td>

**[Pify][]** 3.0.0.<br>
[Sindre Sorhus][sindresorhus] &c.<br>
2015–2018.<br>
MIT License.

<td>

```js
pify(fs.readFile)('package.json', 'utf8')
  .then(data => {
    console.log(JSON.parse(data).name)
  })
```

<td>

```js
'package.json'
  |> await pify(fs.readFile)(#, 'utf8')
  |> JSON.parse |> #.name
  |> console.log
```

<td>

<tr>
<td>

**[Fetch Standard][]**.<br>
[Anne van Kesteren][annevk] &c.<br>
2011–2018.<br>
WHATWG.<br>
Creative Commons BY.

<td>

```js
fetch('/music/pk/altes-kamuffel')
  .then(res => res.blob())
  .then(playBlob)
```

<td>

```js
'/music/pk/altes-kamuffel'
  |> await fetch(#)
  |> await #.blob()
  |> playBlob
```

<td>

<tr>
<td>″″
<td>

```js
fetch('https://example.com/',
  { method: 'HEAD' }
).then(res =>
  log(res.headers.get('content-type'))
)
```

<td>

```js
'https://example.com/'
  |> await fetch(#, { method: 'HEAD' })
  |> #.headers.get('content-type')
  |> log
```

<td>

<tr>
<td>″″
<td>

```js
fetch('https://pk.example/berlin-calling',
  { mode: 'cors' }
).then(response => {
  if (response.headers.get('content-type')
    ??.toLowerCase()
    .indexOf('application/json') >= 0
  ) {
    return response.json()
  } else {
    throw new TypeError()
  }
}).then(processJSON)
```

<td>

```js
const response = 'https://pk.example/berlin-calling'
  |> await fetch(#, { mode: 'cors' });
response
  |> #.headers.get('content-type')
  |> #??.toLowerCase()
  |> #.indexOf('application/json')
  |> # >= 0
  |> # ? response : throw new TypeError()
  |> await #.json()
  |> processJSON
```

<td>

</tbody>

</table>

Nested expressions like those in the second column occur often in JavaScript. They occur whenever any single value must be processed by a series of transformations, whether they be operations, functions, or constructors. Unfortunately, this nested expression—and many like it—can be quite messy spaghetti code, due to its mixing of prefix, infix, and postfix expressions together. Writing the code requires many nested levels of indentation. Reading the code requires checking both the left and right of each subexpression to understand its data flow.

```js
new User.Message(
  capitalize(
    doubledSay(
      (await stringPromise)
        ?? throw new TypeError(`Expected string from ${stringPromise}`)
    )
  ) + '!'
)
```

With the smart pipe operator, the code above could be terser and, literally, straightforward. Prefix, infix, and postfix expressions would be less tangled together in threads of spaghetti. Instead, data values would be piped from left to right through a single flat thread of postfix expressions, essentially forming a [Reverse Polish notation][]. The syntax statically is [term rewritable into already valid code][term rewriting] with theoretically zero runtime cost.

```js
stringPromise
  |> await #
  |> # ?? throw new TypeError()
  |> doubleSay(#, ', ')
  |> capitalize // a tacit unary function call
  |> # + '!'
  |> new User.Message // a tacit unary constructor call
```

This code’s terseness and flatness may be both easier for the JavaScript programmer to read and to edit. The reader may follow the flow of data more easily through this single flattened thread of postfix operations. And the editor may more easily add or remove operations at the beginning, end, or middle of the thread, without changing the indentation of many nearby, unrelated lines.

Similar use cases appear numerous times in JavaScript code, whenever any value is transformed by expressions of any type: function calls, property calls, method calls, object constructions, arithmetic operations, logical operations, bitwise operations, `typeof`, `instanceof`, `await`, `yield` and `yield *`, and `throw` expressions. In particular, the styles of [functional programming][], [dataflow programming][], and [tacit programming][] may benefit from pipelining. The smart pipe operator can simply handle them all.

Note also that it was not necessary to include parentheses for `capitalize` or `new User.Message`; they were implicitly included as a unary function call and a unary constructor call, respectively. That is, the preceding example is equivalent to:
```js
'hello'
  |> await #
  |> # ?? throw new TypeError(`Expected string from ${#}`)
  |> doubleSay(#, ', ')
  |> capitalize(#)
  |> # + '!'
```

Being able to automatically detect this [tacit style][] is the **smart** part of this “smart pipe operator”.

## Nomenclature
The binary operator itself `|>` may be referred to as a **pipe**, a **pipe operator**, or a **pipeline operator**; all these names are equivalent. This specification will prefer the term “pipe operator”.

A pipe operator between two expressions forms a **pipe expression**. One or more pipe expressions in a chain form a **pipeline**.

For each pipe expression, the expression before the pipe is the pipeline’s **head**. A pipeline’s head may also be called its **left-hand side (LHS)**, because it’s left to the pipe. (The head could also be referred to as the pipe’s **topic expression**, the pipe’s **[antecedent][]**, or the pipe’s **[binder][binding]**.) The *value* to which the head evaluates may be referred to as the **topic value**, **head value**, or **LHS value**.

The expression after a pipe is the pipeline’s **body**. A pipeline’s body may also be called its **right-hand side (RHS)**, because it’s to the right of the pipe. When the body is evaluated according to its [runtime semantics][], that value may be referred to the **pipeline’s value**.

Where “pipeline” is used as a verb, the pipe operator is said **to pipeline its topic through its body**, resulting in the pipeline’s value.

A **pipeline-level expression** is an expression at the same [ level level of the pipe operator][operator precedence]. Although all pipelines are pipeline-level expressions, most pipeline-level expressions are not actually pipelines. Conditional operations, logical-or operations, or any other expressions that have tighter [operator precedence][] than the pipe operation—those are also pipeline-level expressions.

The special token `#` is a **topic anaphor**, a **topic bindee**, or **topic placeholder**: it is a nullary operator that acts like a special variable: implicitly bound to its value, but still lexically scoped.

The topic anaphor could also be called a “**topic variable**”, but this phrase is a misnomer. The topic anaphor is *not* actually a variable identifier and cannot be manually declared (`const #` is a syntax error), nor can it be assigned with a value (`# = 3` is a syntax error). Instead, the topic anaphor is implicitly, lexically bound only within pipeline bodies.

When a pipeline’s body is in this **anaphoric style**, it itself can be also referred to as an **anaphora**. A pipeline body in anaphoric style forms an inner lexical scope—called the pipeline’s **anaphoric scope**—within which the topic anaphor is implicitly bound to the value of the topic, acting as a **placeholder** for the topic’s value.

Alternatively, you may omit the topic anaphors entirely, if the body is just a **simple reference** to a function or constructor, such as with `… |> capitalize` and `… |> new User.Message`. Such a pipeline body is in **tacit style**. [Tacit style][] is described in more detail later.

### Explanation of conventions
The term [“**topic**” comes from linguistics][topic and comment] and have precedent in prior programming languages’ use of “topic variables”.

The terms “**topic expression**” and “**antecedent**” are preferred instead of “**LHS**” because, in the future, the [topic concept could be extended to other syntax][possible future extensions to topic anaphora], not just pipelines. However, “LHS” is still a fine and acceptable name for topic expression in the pipeline operator.

The term “**topic anaphor**” is preferred to the phrase “**topic variable**” because the latter is a misnomer. The topic anaphor is *not* a variable identifier. Unlike variables, it cannot be manually declared (`const #` is a syntax error), nor can it be assigned with a value (`# = 3` is a syntax error).

“Topic anaphor” is also preferred to “**topic placeholder**”, to avoid confusion with the placeholders of another TC39 proposal—[syntactic partial application][]. These placeholders (currently denoted by nullary `?`) are of a different nature than topic anaphors. Instead of referring to a single value bound earlier in the surrounding lexical context, these **parameter placeholders** act as the parameter to a new function. When this new function is called, those parameter placeholders will be bound to multiple argument values.

The term “**body**” is preferred instead of “**RHS**” because “topic” is preferred to “LHS”. However, “RHS” is still a fine and acceptable name for the body of the pipeline operator.

## Grammar
This grammar of the pipeline operator juxtaposes brief rules written for the JavaScript programmer with formally written changes to the ES standard. The formal changes are hidden within [TO DO: what’s a good phrase for `<details>`?].

This proposal uses the [same grammar notation as that from the ES standard][ES grammar notation].

### Lexing
The smart pipe operator adds two new tokens to JavaSCript: `|>` the binary pipe, and `#` the topic anaphor.

<details>
<summary>Formal grammar details</summary>

The lexical rule for [punctuator tokens][] would be modified so that `|>` and `#` are added:
```
Punctuator :: one of
  `{` `(` `)` `[` `]` `.` `...` `;` `,` `<` `>` `<=` `>=` `==` `!=` `===` `!==`
  `+` `-` `*` `%` `**` `++` `--`
  `<<` `>>` `>>>` `&` `|` `^` `!` `~` `&&` `||` `?` `:` `|>` `#`
  `=` `+=` `-=` `*=` `%=` `**=` `<<=` `>>=` `>>>=` `&=` `|=` `^=` `=>`
```

</details>

### Grammar parameters
<details>
<summary>Formal grammar details</summary>

In the ES standard, expressions in general are parameterized with three flags:

* **`In`**: Whether the current context allows the [`in` relational operator][], which is false only in the headers of [`for` iteration statements][].
* **`Yield`**: Whether the current context is within a `yield` expression/declaration.
* **`Await`**: Whether the current context is within an `await` expression/declaration.

</details>

### Operator precedence
As a binary operation forming compound expressions, the [operator precedence and associativity][MDN’s guide on expressions and operators] of pipelining must be determined, relative to other operations.

Precedence is tighter than assignment (`=`, `+=`, …), generator `yield` and `yield *`, and sequence `,`; and it is looser than every other type of expression. See also [MDN’s guide on expressions and operators][]. If the pipe operation were any tighter than this level, its body would have to be parenthesized for many frequent types of expressions. However, the result of a pipeline is also expected to often serve as the body of a variable assignment `=`, so it is tighter than assignment operators.

The pipe operator actually has [bidirectional association][]. However, for the purposes of this grammar, it will

The following table shows how the topic reference `#` and the pipe operator `|>` are integrated into the hierarchy of operators. All expression levels in JavaScript from **tightest to loosest**. Each level includes all the expression types listed for that level—**as well as** any expression types from any precedence level that is listed **above** it.

| Precedence level     | Type                          | Form             | Associativity / fixity   |
|----------------------|-------------------------------|------------------|--------------------------|
| Primary level        | This references               | `this`           |                          |
| ″″                   | **Topic references**          | **`#`**          |                          |
| ″″                   | Identifier references         | `a` …            |                          |
| ″″                   | Null literals                 | `null`           |                          |
| ″″                   | Boolean literals              | `true` `false`   |                          |
| ″″                   | Numeric literals              | `0` …            |                          |
| ″″                   | Array literals                | `[…]`            | circumfix                |
| ″″                   | Object literals               | `{…}`            | circumfix                |
| ″″                   | Function literals             |                  |                          |
| ″″                   | Class expressions             | `class … {…}`    | circumfix                |
| ″″                   | Generator expressions         |                  |                          |
| ″″                   | Async-function expressions    |                  |                          |
| ″″                   | Regular-expression literals   | `/…/`…           | circumfix                |
| ″″                   | Template literals             | ```…```          | circumfix                |
| ″″                   | Parenthesized expressions     | `(…)`            | circumfix                |
| LHS level            | Dynamic properties            | `…[…]`           | LTR infix with circumfix |
| ″″                   | Static properties             | `….…`            | LTR infix                |
| ″″                   | Tagged templates              | ``…`…```         | LTR infix                |
| ″″                   | Super properties              | `super.…`        | LTR infix                |
| ″″                   | Meta properties               | `meta.…`         | unchainable prefix       |
| ″″                   | Super call operations         | `super(…)`       | unchainable prefix       |
| ″″                   | New operations                | `new …`          | RTL prefix               |
| ″″                   | Normal call expressions       | `…(…)`           | LTR infix with circumfix |
| Postfix unary level  | Postfix increment op.s        | `…++`            | postfix                  |
| ″″                   | Postfix decrement op.s        | `…--`            | postfix                  |
| Prefix unary level   | Prefix increment op.s         | `++…`            | RTL prefix               |
| Prefix unary level   | Prefix decrement op.s         | `--…`            | ″″                       |
| ″″                   | Delete op.s                   | `delete …`       | ″″                       |
| ″″                   | Void op.s                     | `void …`         | ″″                       |
| ″″                   | Unary `+`/`-` op.s            | ″″               | ″″                       |
| ″″                   | Bitwise NOT op.s `~…`         | ″″               | ″″                       |
| ″″                   | Logical NOT op.s `!…`         | ″″               | ″″                       |
| Exponentiation level | Exponentiation op.s           | `… ** …`         | RTL infix                |
| Multiplicative level | Multiplication op.s           | `… * …` op.s     | LTR infix                |
| ″″                   | Division op.s                 | `… / …`          | ″″                       |
| ″″                   | Modulus op.s                  | `… % …`          | ″″                       |
| Additive level       | Addition op.s                 | `… + …`          | ″″                       |
| ″″                   | Subtraction op.s              | `… - …`          | ″″                       |
| Bitwise-shift level  | Left-shift op.s               | `… << …`         | ″″                       |
| ″″                   | Right-shift op.s              | `… >> …`         | ″″                       |
| ″″                   | Signed right-shift op.s       | `… >> …`         | ″″                       |
| Relational level     | Greater-than op.s             | `… < …`          | ″″                       |
| ″″                   | Less-than op.s                | `… > …`          | ″″                       |
| ″″                   | Greater-than-or-equal-to op.s | `… >= …`         | ″″                       |
| ″″                   | Less-than-or-equal-to op.s    | `… <= …`         | ″″                       |
| ″″                   | Containment op.s              | `… in …`         | ″″                       |
| ″″                   | Instance-of op.s              | `… instanceof …` | ″″                       |
| Equality level       | Abstract equality op.s        | `… == …`         | ″″                       |
| ″″                   | Abstract inequality op.s      | `… != …`         | ″″                       |
| ″″                   | Strict equality op.s          | `… === …`        | ″″                       |
| ″″                   | Strict inequality op.s        | `… !== …`        | ″″                       |
| Bitwise-AND level    |                               | `… & …`          | ″″                       |
| Bitwise-XOR level    |                               | `… ^ …`          | ″″                       |
| Bitwise-OR level     |                               | `… | …`          | ″″                       |
| Logical-AND level    |                               | `… ^^ …`         | ″″                       |
| Logical-OR level     |                               | `… || …`         | ″″                       |
| Conditional level    |                               | `… ? … : …`      | RTL ternary infix        |
| **Pipeline level**   |                               | **`… \|> …`**    | LTR infix                |
| Assignment level     | Arrow functions               | `… => …`         | RTL infix                |
| ″″                   | Async arrow functions         | `async … => …`   | RTL infix                |
| ″″                   | Reference-assignment op.s     | `… = …`          | ″″                       |
| ″″                   |                               | `… += …`         | ″″                       |
| ″″                   |                               | `… -= …`         | ″″                       |
| ″″                   |                               | `… *= …`         | ″″                       |
| ″″                   |                               | `… %= …`         | ″″                       |
| ″″                   |                               | `… **= …`        | ″″                       |
| ″″                   |                               | `… <<= …`        | ″″                       |
| ″″                   |                               | `… >>= …`        | ″″                       |
| ″″                   |                               | `… >>>= …`       | ″″                       |
| ″″                   |                               | `… &= …`         | ″″                       |
| ″″                   |                               | `… |= …`         | ″″                       |
| Yield level          |                               | `yield …`        | RTL prefix               |
| ″″                   |                               | `yield * …`      | ″″                       |
| Ultimate level       | Comma op.s                    | `…, …`           | LTR infix                |


The [assignment operator][] is already parameterized on `In`, `Yield`, and `Await`; it may be a conditional expression, yield expression, arrow function, async arrow function, or assignment:
```
// Old version
AssignmentExpression[In, Yield, Await] :
  ConditionalExpression[?In, ?Yield, ?Await]
  [+Yield] YieldExpression[?In, ?Await]
  ArrowFunction[?In, ?Yield, ?Await]
  AsyncArrowFunction[?In, ?Yield, ?Await]
  LeftHandSideExpression[?Yield, ?Await]
    `=` AssignmentExpression[?In, ?Yield, ?Await]
  LeftHandSideExpression[?Yield, ?Await]
    AssignmentOperator AssignmentExpression[?In, ?Yield, ?Await]
```

Here, the conditional-expression rule (`AssignmentExpression : ConditionalExpression`) would be replaced with one for pipeline expressions (`AssignmentExpression : PipelineExpression`):
```
// New version
AssignmentExpression[In, Yield, Await] :
  PipelineExpression[?In, ?Yield, ?Await]
  [+Yield] YieldExpression[?In, ?Await]
  …
```

An expression is a pipeline-level expression (a `PipelineExpression`) only if:

* It is either also a conditional-level expression `ConditionalExpression`, with the same parameters above;
* Or it is another pipeline-level expression, followed by a `|>` token, then a pipeline body (`PipelineBody`, defined next), with the same parameters as above.

```
// New rule
PipelineExpression[In, Yield, Await] :
  ConditionalExpression[?In, ?Yield, ?Await]
  PipelineExpression[?In, ?Yield, ?Await] `|>`
    PipelineBody[?In, ?Yield, ?Await]
```

### Tacit style
Most pipelines will use the topic anaphor `#` in their bodies. But for two certain simple cases—unary functions and constructors—you may omit the anaphor from the body. The body is just a **simple reference** to a function or constructor, such as with `… |> capitalize` and `… |> new User.Message`.

This is called [**tacit style** or **point-free style**][tacit programming]. When a pipe is in tacit style, we refer to the body as a **tacit function** or a **tacit constructor**, depending on the rules in [tacit style][].

The body’s value would then be called as a unary function or constructor, without having to use the topic anaphor as an explicit argument.

**If a pipeline** is of the form **`{{topic}} |> {{identifier}}`** (or `{{topic}} |> {{identifier0}}.{{identifier1}}` or `{{topic}} |> {{identifier0}}.{{identifier1}}.{{identifier2}}` or …), then the pipeline is a **tacit function call**. The **pipeline’s value** is **`{{body}}({{topic}})`**.

**If a pipeline’s `body`** is of the form **`{{topic}} |> new {{identifier}}`** (or `{{topic}} |> new {{identifier0}}.{{identifier1}}` or `{{topic}} |> new {{identifier0}}.{{identifier1}}.{{identifier2}}` or …), then the pipeline is a **tacit constructor call**. The **pipeline’s value** is **`new {{bodyConstructor}}({{topic}})`**, where {{bodyConstructor}} is {{body}} with the `new` removed from its start.

Therefore, a pipeline in **tacit style *never*** has **parentheses `(…)` or brackets `[…]`** in its body. Neither `… |> object.method()` nor `… |> object.method(arg)` nor `… |> object[symbol]` nor `… |> object.createFunction()` are in tacit style (in fact, they all have invalid syntax, due to their being anaphoric style without any anaphora).

The tacit style supports using simple identifiers, possibly with chains of simple property identifiers. If there are any operators, parentheses (including for method calls), brackets, or anything other than identifiers and `.`s, then it cannot be a tacit call; it must be a function call.

|                                                                                   |                                     |                     |                                          |                          |                                 |                      |                                          |                            |
|-----------------------------------------------------------------------------------|-------------------------------------|---------------------|------------------------------------------|--------------------------|---------------------------------|----------------------|------------------------------------------|----------------------------|
| **When a body needs parentheses or brackets**…                                    | ❌ `… \                              | > object.method()`  | ❌ `… \                                   | > object.method(arg)`    | ❌ `… \                          | > object[symbol]`    | ❌ `… \                                   | > object.createFunction()` |
| …then **don’t use tacit style**, and instead **use a topic anaphor** in the body… | `… \                                | > object.method(#)` | `… \                                     | > object.method(arg, #)` | `… \                            | > object[symbol](#)` | `object.createFunction()(#)`             |                            |
| …or **assign the body to a variable**, then **use that variable as a tacit body** | `const method = object::method; … \ | > method`           | `const method = object::method(arg); … \ | > method`                | `const fn = object[symbol]; … \ | > fn`                | `const fn = object.createFunction(); … \ | > fn`                      |

The goal here is to minimize the parsing lookahead that the compiler must check before it can distinguish between tacit style and topic-token style. By restricting the space of valid tacit-style pipeline bodies (that is, without topic anaphors), the rule prevents [garden-path syntax][] that would otherwise be possible: such as `… |> compose(f, g, h, i, j, k, #)`.

The JavaScript programmer is encouraged to use topic anaphors and avoid tacit style, where tacit style may be visually confusing to the reader.

### Anaphoric style
**If a pipeline** of the form `{{topic}} |> {{body}}` is ***not* in tacit style** (that is, it is *not* a tacit function call or tacit constructor call), then it **must be in anaphoric style**. The **pipeline’s value** is **`do { const {{topicIdentifier}} = {{topic}}; {{substitutedBody}} }`**, where:

* `{{topicVariable}}` is any [identifier that is *not* already used by any variable in the outer lexical context or the body’s inner anaphoric context][lexically hygienic],
* And `{{substitutedBody}}` is `{{body}}` but with every instance of outside of the topic anaphor replaced by `{{topicVariable}}`.

Another goal is to help the editing JavaScript programmer: statically preventing them from accidentally omitting a topic anaphor where they meant to put one. For instance, if `x |> 3` were not a syntax error, then it would be a useless operation and almost certainly not what the editor intended.

| Expression                            | Result                          |
| ------------------------------------- | ------------------------------- |
| **`'2018' \|> Date(#)`**              | **`Date('2018')`**              |
| **`'2018' \|> Date`**                 | **`Date('2018')`**              |
|   `'2018' \|> Date()`                 |   syntax error: missing `#`     |
|   `'2018' \|> (Date)`                 |   syntax error: missing `#`     |
| **`'2018' \|> Date.parse(#)`**        | **`Date.parse('2018')`**        |
| **`'2018' \|> Date.parse`**           | **`Date.parse('2018')`**        |
|   `'2018' \|> Date.parse()`           |   syntax error: missing `#`     |
|   `'2018' \|> (Date.parse)`           |   syntax error: missing `#`     |
|   `'2018' \|> Date['parse'](#)`       | **`Date['parse']('2018')`**     |
|   `'2018' \|> Date['parse']()`        |   syntax error: missing `#`     |
|   `'2018' \|> (Date['parse'])`        |   syntax error: missing `#`     |
| **`'2018' \|> global.Date.parse(#)`** | **`global.Date.parse('2018')`** |
| **`'2018' \|> global.Date.parse`**    | **`global.Date.parse('2018')`** |
|   `'2018' \|> global.Date.parse()`    |   syntax error: missing `#`     |
|   `'2018' \|> (global.Date.parse)`    |   syntax error: missing `#`     |
| **`'2018' \|> new global.Date(#)`**   | **`new Date('2018')`**          |
| **`'2018' \|> new global.Date`**      | **`new Date('2018')`**          |
|   `'2018' \|> new global.Date()`      |   syntax error: missing `#`     |
|   `'2018' \|> (new global.Date)`      |   syntax error: missing `#`     |
|   `'2018' \|> Date['parse'](#)`       | **`Date['parse']('2018')`**     |

### Multiple topic anaphors in a pipeline body
The topic anaphor may be used multiple times in a pipeline body. Each use refers to the same value (wherever the topic anaphor is not overridden by another, inner pipeline’s anaphoric scope). Because it is bound to the result of the topic, the topic is still only ever evaluated once. The lines in each of the following are equivalent:
```js
do { const $ = …; f($, $) }
… |> f(#, #)
```
***
```js
do { const $ = …; [$, $ * 2, $ * 3] }
… |> [#, # * 2, # * 3]
```

### Inner functions
Both the topic expression and the body of a pipeline may contain nested inner functions. The lines in the following are equivalent:
```js
do { const $ = …; settimeout(() => $ * 500) }
… |> settimeout(() => # * 500)
```

### Deeply nested pipelines
Both the topic expression and the body of a pipeline may contain nested inner functions. This is not encouraged, but it is still permitted. All lines in the following are equivalent:
```js
do { const $ = …; settimeout(() => f($) * 500) }
do { const $ = …; settimeout(() => f($) |> # * 500) }
do { const $ = …; settimeout(() => $ |> f |> # * 500) }
… |> settimeout(() => f(#) * 500)
… |> settimeout(() => f(#) |> # * 500)
… |> settimeout(() => # |> f |> # * 500)
```

## [TO DO: Remove this section]
A pipeline `topic |> body` is a left-associative (though see [bidirectional associativity][]) binary operation with a loose precedence.



### Smart body syntax
When the parser checks the body/RHS, it must determine what style it is in. There are four outcomes for each pipeline body: tacit constructor, tacit function, anaphora, or syntax error.

```
// New rule
PipelineBody[In, Yield, Await] :
  PipelineTacitFunction
  PipelineTacitConstructor
  PipelineAnaphora[In, Yield, Await]
```

1. If the body is a mere identifier, optionally with a chain of properties, and with no parentheses or brackets, then that identifier is interpreted to be a **tacit function**.
  ```
  PipelineTacitFunction :
    SimpleMemberExpression
  ```

2. If the body starts with `new`, followed by mere identifier, optionally with a chain of properties, and with no parentheses or brackets, then that identifier is interpreted to be a **tacit constructor**.
  ```
  PipelineTacitConstructor :
    `new` SimpleMemberExpression
  ```

3. Otherwise, the body is now an arbitrary expression. Because the expression is not in tacit style, it must use that pipe’s topic anaphor. ([TO DO: Formalize this static semantic as `usesTopic()`.] Topic anaphors from the anaphoric scopes of other, inner pipelines do not count.) If there is no such topic anaphor in the expression, then a syntax error is thrown.

  `PipelineAnaphora` will be defined in the next section.

If a pipeline body *never* uses a topic anaphor, then it must be a permitted tacit unary function (single identifier or simple property chain). Otherwise, it is a syntax error. In particular, tacit style *never* uses parentheses. If they need to have parentheses, then they need to have use the topic anaphor. See also [property-chained tacit style][].

### General semantics
[TO DO: Rewrite this section to the conventions of the ES specification.]

A pipe expression’s semantics are:
1. The topic expression is evaluated into the topic’s value; call this `topicValue`.
2. [The body is tested for its type][smart body syntax]: Is it in tacit style (as a tacit function or a tacit constructor), is it in anaphoric style, or is it an invalid body?

  * If the body is a tacit function (such as `f` and `M.f`):
    1. The body is evaluated (in the current lexical context); call this value the `bodyFunction`.
    2. The entire pipeline’s value is `bodyFunction(topicValue)`.

  * If the body is a tacit constructor (such as `new C` and `new M.C`):
    1. The portion of the body after `new` is evaluated (in the current lexical context); call this value the `BodyConstructor`.
    2. The entire pipeline’s value is `new BodyConstructor(topicValue)`.

  * If the body is in anaphoric style (such as `f(#, n)`, `o[s][#]`, `# + n`, and `#.p`):
    1. An inner lexical context is entered.
    2. Within this inner context, `topicValue` is bound to a variable.
    3. Within this inner context, with the variable binding to `topicValue`, the body is evaluated; call the resulting value `bodyValue`.
    4. The entire pipeline’s value is `bodyValue`.

  * Otherwise, if the body is an invalid body (such as `f()`, `f(n)`, `o[s]`, `+ n`, and `n`), then throw a syntax error explaining that the pipeline’s topic anaphor is missing from its body.

### Term rewriting anaphoric style
Pipe bodies in anaphoric style can be further explained with a nested `do` expression. There are two ways to illustrate this equivalency. The first way is to [replace each pipe expression’s topic anaphors with an autogenerated variable][term rewriting with autogenerated variables], which must be guaranteed to be [lexically hygienic][] and to not conflict with other variables. The alternative way is to [use two variables—the topic anaphor `#` and a single dummy variable][term rewriting with single dummy variable]—which also preserves [lexical hygiene][lexically hygienic].

#### Term rewriting with autogenerated variables
The first way to illustrate the operator’s semantics is to replace each pipe expression’s topic anaphors with an autogenerated variable, which must be guaranteed to not conflict with other variables.

Let us pretend that each pipe expression autogenerates a new, [lexically hygienic][] variable (`#₀`, `#₁`, `#₂`, `#₃`, …), which in turn replaces each topic anaphor `#` in each pipeline body. (These `#ₙ` variables are not true syntax; it is merely for illustrative purposes. You cannot actually assign or use `#ₙ` variables.) Let us also group the expressions with left associativity (although this is arbitrary, because [right associativity would also work][bidirectional associativity]).

With this notation, each line in this example would be equivalent to the other lines.
```js
1 |> # + 2 |> # * 3

// Static term rewriting
(1 |> # + 2) |> # * 3
do { const #₀ = 1; # + 2 } |> # * 3
do { const #₁ = do { const #₀ = 1; # + 2 }; #₁ * 3 }

// Runtime evaluation
do { const #₀ = do { 1 + 2 }; #₀ * 3 }
do { const #₀ = 3; #₀ * 3 }
do { do { 3 * 3 } }
9
```

Consider also the motivating first example above:
```js
stringPromise
  |> await #
  |> # ?? throw new TypeError()
  |> doubleSay // a tacit unary function call
  |> capitalize // also a tacit unary function call
  |> # + '!'
```

Under left associativity, this would be statically equivalent to the following:
```js
do {
  const #₃ = do {
    const #₂ = do {
      const #₁ = do {
        const #₀ = await stringPromise;
        #₀ ?? throw new TypeError()
      };
      doubleSay(#₁)
    };
    capitalize(#₂)
  };
  #₃ + '!'
}
```

In general, for each pipe expression `topic |> body`, assuming that `body` is in anaphoric style, that is, assuming that `body` contains an unshadowed topic anaphor:

* Let `#ₙ` be a [hygienically autogenerated][lexically hygienic] topic anaphor, `#ₙ`, where <var>n</var> is a number that would not conflict with the name of any other autogenerated topic anaphor in the scope of the entire pipe expression.
* Also let `substitutedBody` be `body` but with all instances of `#` replaced with `#ₙ`.
* Then the static term rewriting (left associative and inside to outside) would simply be: `do { const #ₙ = topic; substitutedBody }`. This `do` expression would act as at the anaphoric scope.

#### Term rewriting with single dummy variable
The other way to demonstrate anaphoric style is to use two variables: the topic anaphor `#` and single [lexically hygienic][] dummy variable `•`. It should be noted that `const # = …` is not a valid statement under this proposal’s actual syntax; likewise, `•` is not a part of the proposal’s syntax. Both forms are for illustrative purposes here only.

With this notation, no variable autogeneration is required; instead, the nested `do` expressions will redeclare the same variables `#` and `•`, shadowing the external variables of the same name as needed. The number example above becomes the following. Each line is still equivalent to the other lines.
```js
1 |> # + 2 |> # * 3

// Static term rewriting
(1 |> # + 2) |> # * 3
do { const • = 1; do { const # = •; # + 2 } } |> # * 3
do { const • = (do { const • = 1; do { const # = •; # + 2 } }); do { const # = •; # * 3 } }

// Runtime evaluation
do { const • = do { do { const # = 1; # + 2 } }; do { const # = •; # * 3 } }
do { const • = do { do { const 1 + 2 } }; do { const # = •; # * 3 } }
do { const • = 3; do { const # = •; # * 3 } }
do { do { const # = 3; # * 3 } }
do { do { 3 * 3 } }
9
```

Consider also the motivating first example above:
```js
stringPromise
  |> await #
  |> # ?? throw new TypeError()
  |> doubleSay // a tacit unary function call
  |> capitalize // also a tacit unary function call
  |> # + '!'
```

Under left associativity, this would be statically equivalent to the following:
```js
do {
  const • = do {
    const • = do {
      const • = do {
        const • = await stringPromise;
        do { const # = •; # ?? throw new TypeError() }
      };
      do { const # = •; doubleSay(#) }
    };
    do { const # = •; capitalize(#) }
  };
  do { const # = •; # + '!' }
}
```

For each pipe expression, evaluated left associatively and inside to outside, the steps of the computation would be:

1. The topic expression is first evaluated in the current lexical context.
2. The topic’s result is bound to a hidden special variable `•`.
3. In a new inner lexical context (the anaphoric scope), the value of `•` is bound to the topic anaphor `#`.
4. The pipe’s body is evaluated within this inner lexical context.
5. The pipe’s result is the result of the body.

### Bidirectional associativity
The pipe operator is presented above as a left-associative operator. However, it is theoretically [bidirectionally associative][associative property]: how a pipeline’s expressions are particularly grouped is functionally arbitrary. One could force right associativity by parenthesizing a pipeline, such that it itself becomes the body of another, outer pipeline.

Consider the above example `1 |> # + 2 |> # * 3`, whose terms were statically rewritten using left associativity and autogenerated, [lexically hygienic][] variables.
```js
// With left associativity and autogenerated hygienic variables.
1 |> # + 2 |> # * 3

// Static term rewriting
(1 |> # + 2) |> # * 3
do { const #₀ = 1; # + 2 } |> # * 3
do { const #₁ = do { const #₀ = 1; # + 2 }; #₁ * 3 }

// Runtime evaluation
do { const #₀ = do { 1 + 2 }; #₀ * 3 }
do { const #₀ = 3; #₀ * 3 }
do { do { 3 * 3 } }
9
```

But if right associativity is forced with `1 |> (# + 2 |> # * 3)`, then the result would be the same: `9`:
```js
// With right associativity and autogenerated hygienic variables.
1 |> # + 2 |> # * 3

// Static term rewriting
1 |> (# + 2 |> # * 3)
1 |> do { const #₀ = # + 2; #₀ * 3 }
do { const #₁ = 1; do { const #₀ = #₁ + 2; #₀ * 3 } }

// Runtime evaluation
do { do { const #₀ = 1 + 2; #₀ * 3 } }
do { do { const #₀ = 3; #₀ * 3 } }
do { do { 3 * 3 } }
9
```

Similarly, `1 |> # + 2 |> # * 3` was also statically term rewritten using a different method: under left associativity and a single dummy variable.
```js
// With left associativity and single dummy variable.
1 |> # + 2 |> # * 3

// Static term rewriting
(1 |> # + 2) |> # * 3
do { const • = 1; do { const # = •; # + 2 } } |> # * 3
do { const • = (do { const • = 1; do { const # = •; # + 2 } }); do { const # = •; # * 3 } }

// Runtime evaluation
do { const • = do { do { const # = 1; # + 2 } }; do { const # = •; # * 3 } }
do { const • = do { do { const 1 + 2 } }; do { const # = •; # * 3 } }
do { const • = 3; do { const # = •; # * 3 } }
do { do { const # = 3; # * 3 } }
do { do { 3 * 3 } }
9
```

If right associativity is forced with `1 |> (# + 2 |> # * 3)`, then the result would be the same: `9`:
```js
// With right associativity and single dummy variable.
1 |> # + 2 |> # * 3

// Static term rewriting
1 |> (# + 2 |> # * 3)
1 |> do { const • = # + 2; do { const # = •; # * 3 } }
do { • = 1; do { const # = •; do { const • = # + 2; do { const # = •; # * 3 } } } }

// Runtime evaluation
do { do { const # = 1; do { const • = # + 2; do { const # = •; # * 3 } } }
do { do { do { const • = 1 + 2; do { const # = •; # * 3 } } }
do { do { do { const • = 3; do { const # = •; # * 3 } } }
do { do { do { do { const # = 3; # * 3 } } }
do { do { do { do { 3 * 3 } } }
9
```

## Static semantics
[TO DO: Brief explanation of static semantics]

### Early errors

### Contains?
[TO DO]

### Is function definition?
[TO DO]

### Is valid simple assignment target?
[TO DO]

### Is

## Runtime semantics
[TO DO]

## Possible future extensions to topic anaphora

## Alternative solutions explored
There are a number of other ways of potentially accomplishing the above use cases. However, the authors of this proposal believe that the smart pipe operator may be the best choice. [TO DO]

## Relation to existing work
[TO DO]

[TO DO: Include commentary on why “topic anaphor” instead of “placeholder”—because verbally confusing with partial-application placeholders—and because forward compatibility with extending the topic concept to other syntax, such as `for { … }`, `matches { … }`, and `given (…) { … }`.]

[`for` iteration statements]: https://tc39.github.io/ecma262/#sec-iteration-statements

[`in` relational operator]: https://tc39.github.io/ecma262/#sec-relational-operators

[annevk]: https://github.com/annevk

[antecedent]: https://en.wikipedia.org/wiki/Antecedent_(grammar)

[assignment operator]: https://tc39.github.io/ecma262/#sec-assignment-operators

[associative property]: https://en.wikipedia.org/wiki/Associative_property

[bidirectional associativity]: #bidirectional-associativity

[binding]: https://en.wikipedia.org/wiki/Binding_(linguistics)

[Clojure pipe]: https://clojuredocs.org/clojure.core/as-%3E

[concatenative programming]: https://en.wikipedia.org/wiki/Concatenative_programming_language

[Daniel “littledan” Ehrenberg of Igalia]: https://github.com/littledan

[dataflow programming]: https://en.wikipedia.org/wiki/Dataflow_programming

[Elixir pipe]: https://elixir-lang.org/getting-started/enumerables-and-streams.html

[Elm pipe]: http://elm-lang.org/docs/syntax#infix-operators

[ES grammar notation]: https://tc39.github.io/ecma262/#sec-syntactic-and-lexical-grammars

[F# pipe]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/functions/index#function-composition-and-pipelining

[formal grammar]: #grammar

[functional programming]: https://en.wikipedia.org/wiki/Functional_programming

[garden-path syntax]: https://en.wikipedia.org/wiki/Garden_path_sentence

[GitHub issue tracker]: https://github.com/tc39/proposal-pipeline-operator/issues

[Hack pipe]: https://docs.hhvm.com/hack/operators/pipe-operator

[jashkenas]: https://github.com/jashkenas

[Julia pipe]: https://docs.julialang.org/en/stable/stdlib/base/#Base.:|>

[lexically hygienic]: https://en.wikipedia.org/wiki/Hygienic_macro

[littledan invitation]: https://github.com/tc39/proposal-pipeline-operator/issues/89#issuecomment-363853394

[LiveScript pipe]: http://livescript.net/#operators-piping

[MDN’s guide on expressions and operators]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Expressions_and_Operators

[mindeavor]: https://github.com/gilbert

[Node-stream piping]: https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options

[nomenclature]: #nomenclature

[OCaml pipe]: http://blog.shaynefletcher.org/2013/12/pipelining-with-operator-in-ocaml.html

[Perl 6 pipe]: https://docs.perl6.org/language/operators#infix_==&gt;

[Pify]: https://github.com/sindresorhus/pify

[pipeline syntax]: #pipeline-syntax

[possible future extensions to topic anaphora]: #possible-future-extensions-to-topic-anaphora

[previous pipe-operator proposal]: https://github.com/tc39/proposal-pipeline-operator

[previous pipeline-placeholder discussions]: https://github.com/tc39/proposal-pipeline-operator/issues?q=placeholder

[Prior pipeline proposal]: https://github.com/tc39/proposal-pipeline-operator/blob/37119110d40226476f7af302a778bc981f606cee/README.md

[property-chained tacit style]: #property-chained-tacit-style

[Proposal 4: Smart Mix on the pipe-proposal wiki]: https://github.com/tc39/proposal-pipeline-operator/wiki#proposal-4-smart-mix

[punctuator tokens]: https://tc39.github.io/ecma262/#sec-punctuators

[reverse Polish notation]: https://en.wikipedia.org/wiki/Reverse_Polish_notation

[R pipe]: https://cran.r-project.org/web/packages/magrittr/index.html

[runtime semantics]: #runtime-semantics

[sindresorhus]: https://github.com/sindresorhus

[smart body syntax]: #smart-body-syntax

[syntactic partial application]: https://github.com/tc39/proposal-partial-application

[tacit programming]: https://en.wikipedia.org/wiki/Tacit_programming

[tacit style]: #tacit-style

[TC39 process]: https://tc39.github.io/process-document/

[term rewriting]: https://en.wikipedia.org/wiki/Term_rewriting

[term rewriting anaphoric style]: #term-rewriting-anaphoric-style

[term rewriting with autogenerated variables]: #term-rewriting-with-single-dummy-variable

[term rewriting with autogenerated variables]: #term-rewriting-with-single-dummy-variable

[term rewriting with single dummy variable]: #term-rewriting-with-single-dummy-variable

[topic and comment]: https://en.wikipedia.org/wiki/Topic_and_comment

[topic variables in other languages]: https://rosettacode.org/wiki/Topic_variable

[Underscore.js]: http://underscorejs.org

[Unix pipe]: https://en.wikipedia.org/wiki/Pipeline_(Unix

[Fetch Standard]: https://fetch.spec.whatwg.org

[WHATWG-stream piping]: https://streams.spec.whatwg.org/#pipe-chains

[operator precedence]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence

[operator precedence (MDN)]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence
