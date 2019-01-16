# ECMAScript proposal: chainable do sugar
For purposes of this document, a “chainable” is any value that: 
1. Is a wrapper around another value (which can be _anything_).
2. Has a `.map(value => value + 1)` method which takes a function that modifies the wrapped value.
3. Has a `.chain(value => Wrapper(value + 1))`  method which takes a function that returns another wrapper (must be the same wrapper type), and “unwraps” the nested wrapper.

Chainables are a powerful programming pattern that can be used to express a large number of pure solutions to programming problems including but not limited to: asynchronous code, safe null checks, dependency injection, state changes, observables, error handling, and lazy evaluation.

## Motivation:
```js
const foo = {bar: null}
const baz = 
  option(foo)
    .chain(foo =>
      option(foo.bar).chain(bar => 
        option(bar.baz).chain(baz =>
          option(baz.biz).map(biz =>
            biz
          )
        )
      )
    )
```

Nested chainable calls can quickly become verbose and clunky to deal with. Async/await helps with this, but is specialized to Promises.

The intent behind this nested sequence of code can be expressed clearer with just a small amount of syntax sugar:

```
const fooOption = option({bar: null})
const baz = 
  do {
    foo <- fooOption
    bar <- option(foo.bar)
    baz <- option(bar.baz)
    biz <- option(baz.biz)

    biz
  }
```

## Sugar:
The rules for the syntax are rather simple:

Anytime a `<-` expression is inside a `do` block, it’s split into two sides. The RHS value has a `chain` method invoked, and the LHS is presented as the sole parameter to the `chain` method. 
For the last `<-` expression, `chain` is replaced with `map`, and the very last expression of the do block is returned from this method

```
do {
  {paramA} <- {exprA}
  {paramB} <- {exprB}

  {exprC}
}
```
Is de-sugared to:
```
{exprA}.chain({paramA} => {
  return {exprB}.map({paramB} => {
    return {exprC}
  })
})
```

## Generalization:
Since the only requirements for this are that a value has a `.chain` and `.map` method, it can work for all kinds of chainables included but not limitied to:

Asynchronous chainables:
```
do {
  user   <- fetchUser('paul')
  img    <- fetchImage(user.profileImage)
  friend <- fetchUser(user.friend_id)

  {...user, img, friend}
}
```

Option (Maybe) chainables:
```
const baz = 
  do {
    posts     <- option(user.posts)
    firstPost <- option(posts[0])

    firstPost.likeCount / posts.length
  }
```

Error handling chainables:
```
const baz = 
  do {
    count1 <- Try(() => parseInt('15'))
    count2 <- Try(() => parseInt('30'))

    count1 * count2
  }
```

These are just some examples. There are many more!

## Example Implementation
[This babel plugin](https://github.com/pfgray/babel-plugin-monadic-do) implements this proposal with a few caveats: 

1. `<<` is used instead of `<-`.
2. A destructuring expression doesn’t work as the LHS of a `<<` expression.

These caveats are simply because this is implemented as a babel plugin. An update to babel would be needed to add the `<-` operator and support destructuring on the LHS.

## Prior Art
Scala has [for comprehensions](https://docs.scala-lang.org/tutorials/FAQ/yield.html)
```scala
for {
   n1 <- fetchNumber(...)
   n2 <- fetchNumber(...)
   n3 <- fetchNumber(...)
} yield (n1 + n2 + n3)
```

Haskell has [do notation](https://en.wikibooks.org/wiki/Haskell/do_notation)
```haskell
do { x1 <- action1
   ; x2 <- action2
   ; mk_action3 x1 x2 }
```
