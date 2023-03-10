+++
date = 2023-03-07T21:12:14+05:30
title = 'Custom marshalers in Go: An unexpected gotcha'
description = 'Catching subtle bugs in struct embedding'
tags = ['go', 'compilers']
+++

There's a very common question that Go devs hear regularly - **_“Is Go object oriented?”_**

And the answer?

<p style="text-align:center">
  <figure>
    <img alt="Well yes, but no" src="./yes-but-no.jpg" height="70%" width="70%" />
    <figcaption style="font-style:italic;font-size:80%;">
      Source: <a href="https://knowyourmeme.com/memes/well-yes-but-actually-no">Know Your Meme</a>
    </figcaption>
  </figure>
</p>

This is what the [Go FAQ](https://go.dev/doc/faq#Is_Go_an_object-oriented_language) has to say about this:

> Yes and no. Although Go has types and methods and allows an object-oriented style of programming, there is no type hierarchy. The concept of “interface” in Go provides a different approach that we believe is easy to use and in some ways more general. There are also ways to embed types in other types to provide something analogous—but not identical—to subclassing. Moreover, methods in Go are more general than in C++ or Java: they can be defined for any sort of data, even built-in types such as plain, “unboxed” integers. They are not restricted to structs (classes).

As a language, Go decided to keep the good parts of object-oriented programming while eliminating the more cumbersome bits. This resulted in a unique (and sometimes unidiomatic) philosophy of programming. Instead of classes, Go provides **receiver functions** - a function that can be attached to a data type and called only by a variable of that specific type. Receiver functions are also unique for values and references, i.e., `func (o Object)` and `func (o *Object)` are independent, allowing us to call different methods depending on whether the caller is a value or a reference. This allows us to emulate classes (to an extent) by having some form of coupling between types and methods.

## How does Go handle inheritance?

Since there is no solid idea of a class, composition is used over inheritance. This is done through **struct embedding**. Just add your child struct type to the parent struct without associating a field name to it. The child struct members are now members of the parent struct too.

```go
type Child struct {
    A int
    B string
}

type Parent struct {
    Child
    // Now, A and B are available to the caller as Parent.A and Parent.B
    C int
    D string
}
```

But what about the receiver functions attached to the child? This is where things get messy. To provide inheritance-style reuse of struct methods, Go uses **method promotion**. The way this is supposed to work is very obvious - all child methods are promoted to the parent unless there is a namespace conflict; on conflict, the parent method is given priority over the child. But there is a third player in Go - the default functions which are called when there is no custom implementation. `MarshalJSON()` is an excellent example of this.

## Understanding JSON marshaling

When `json.Marshal(foo)` is called, Go looks for a `MarshalJSON()` receiver function implemented by the type of `foo`. If it does not exist, then it calls the default `MarshalJSON()` method. The same applies to `json.Unmarshal(foo)` with `UnmarshalJSON()`. For brevity's sake, I'll use marshaling to explain concepts in the rest of the article, but everything is valid for unmarshaling too.

Continuing from the above code, try to guess what happens in the following situation:

```go
func (c *Child) MarshalJSON() ([]byte, error) {
    return []byte(`{"Child":"Hello there"}`), nil
}

func main() {
    p := Parent{
        A: 1,
        B: "2",
        C: 3,
        D: "4",
    }
    // json.MarshalIndent is the pretty cousin of json.Marshal
    j, _ := json.MarshalIndent(&p, "", "  ")
    fmt.Println(string(j))
}
```

You probably expected the output to look like this:

```json
{
  "Child": "Hello there",
  "C": 3,
  "D": "4"
}
```

But instead, the output is:

```json
{
  "Child": "Hello there"
}
```

What? Where did the C and D fields go (heh, Go)? There may be an answer in the language specification for struct embedding. Here's what the [Go spec](https://go.dev/ref/spec#Struct_types) says:

> Given a struct type S and a named type T, promoted methods are included in the method set of the struct as follows:
>
> - If S contains an embedded field T, the method sets of S and *S both include promoted methods with receiver T. The method set of *S also includes promoted methods with receiver \*T.
> - If S contains an embedded field *T, the method sets of S and *S both include promoted methods with receiver T or \*T.

Right. In short, for an embedded type `T`:

- The methods on `T` is promoted to both the value and pointer of the parent type.
- If `T` is embedded (by value), then the methods on `*T` are promoted only to `*S`.
- If `*T` is embedded (by reference), then the methods on `*T` are promoted to both `S` and `*S`.

Now it's starting to make some sense. Since `Child` has a custom `MarshalJSON(),` this is promoted to `Parent`, overriding the default `MarshalJSON().` And clearly, `Child`’s marshaler will have no clue about any of `Parent`'s members.

## Oh no, make it work!

So how do we deal with this mess? How about implementing a dummy marshaler for `Parent`, which calls the default method within?

```go
func (p *Parent) MarshalJSON() ([]byte, error) {
    return json.Marshal(p)
}
```

When you run this, something wonderful happens - a stack overflow. Why did everything just go up in flames? To answer that, we must understand how `MarshalJSON()` works under the hood. Let's explore the [`encode.go`](https://cs.opensource.google/go/go/+/refs/tags/go1.20.1:src/encoding/json/encode.go) file in the `encoding/json` package.

- At line [161](https://cs.opensource.google/go/go/+/refs/tags/go1.20.1:src/encoding/json/encode.go;l=161;drc=e7f2e5697ac8b9b6ebfb3e0d059a8c318b4709eb), `Marshal()` calls `e.marshal()`.
- At line [330](https://cs.opensource.google/go/go/+/refs/tags/go1.20.1:src/encoding/json/encode.go;l=330;drc=e7f2e5697ac8b9b6ebfb3e0d059a8c318b4709eb), `marshal()` calls `e.reflectValue()`, which identifies the type at runtime and marshals it accordingly.
- At line [358](https://cs.opensource.google/go/go/+/refs/tags/go1.20.1:src/encoding/json/encode.go;l=358;drc=e7f2e5697ac8b9b6ebfb3e0d059a8c318b4709eb), `reflectValue()` calls `valueEncoder()`, which returns an encoder function for the type.
- At line [477](https://cs.opensource.google/go/go/+/refs/tags/go1.20.1:src/encoding/json/encode.go;l=477;drc=e7f2e5697ac8b9b6ebfb3e0d059a8c318b4709eb), `valueEncoder()` calls `MarshalJSON()`
- In our code, `MarshalJSON()` calls `json.Marshal()`.

There we go! `MarshalJSON()` calls `json.Marshal()`, which calls `MarshalJSON()`, which calls... you know where this is going. They just call each other until we run out of memory and crash. So what is the workaround for this? We can use a [type alias](https://go.dev/ref/spec#Alias_declarations) and call `json.Marshal()` on that instead to prevent the infinite recursion that was happening previously.

```go
func (p *Parent) MarshalJSON() ([]byte, error) {
    type Alias Parent
    a = (*Alias)(p)
    return json.Marshal(a)
}
```

But the result of running this function is even more unexpected:

```json
{
  "A": 1,
  "B": "2",
  "C": 3,
  "D": "4"
}
```

It looks like all the fields of `Parent` are present, but where did our custom logic for marshaling `Child` go? Remember, `Child` is embedded in `Parent` as a value, not as a reference (`*Child`); as `MarshalJSON()` is attached to `c *Child`, it's never called! If we attach this function to `c Child` instead, we're back to square one with the recursion problem. The type aliasing hack won't work here as casting an object instead of the pointer retains the methods attached to the type. Tsk tsk, we've hit a dead end!

## What's the solution?

This problem is an unfortunate side effect of struct embedding and can only be fixed with significant changes to the language implementation. It has been a bone of contention amongst developers, with one person even proposing [removing the entire feature](https://github.com/golang/go/issues/22013) from Go 2.0. This specific problem with JSON has also been discussed at length in [this GitHub issue](https://github.com/golang/go/issues/30148). The only way to solve this is to write a custom marshaler for `Parent` that calls the marshaler for `Child` and manually marshals the remaining fields. As for compliance with the JSON specification, that's a dragon you will have to tame on your own.

## Epilogue

Custom marshalers are common across Go projects and can be a pain to implement with embedded types. Being aware of this issue and the workarounds considerably reduces the time and effort required to implement them correctly.
