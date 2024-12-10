---
date: 2024-12-10T20:19:47+05:30
title: "Oxidised causal inference in software design"
description: "Does greatness come from the art or the artist?"
tags: ['rust']
---
In recent years, Rust has taken off spectacularly, becoming one of the most loved programming languages, and gaining adoption by some of the biggest software companies in the world. One of the main reasons is memory safety - the language is designed such that it is nigh impossible for you to write code that has memory access vulnerabilities (okay, not _impossible_, but definitely much much harder than its contemporaries). Novices tend to bang their head against the wall, sacrifice their hopes and dreams, and exorcise their motherboard in an attempt to satisfy the borrow checker and get their code to compile. Of course, as with any other challenging endeavour, it becomes easier with practice and experience.

Some other reasons include excellent functional patterns in the standard library, a robust type system that allows enforcing invariants across program states, brilliant tooling and ecosystem (cargo, [crates.io](https://crates.io), rust-analyzer), Blazingly Fast™ execution times, the list goes on and on. But a slightly more subtle reason that is typically overlooked is that it teaches you _how to think_ in certain ways that carry across programming languages.

# Do you remember (Earth, Wind, & Fire)?

If you write Rust long enough, thinking about memory semantics becomes second nature. When you have to touch that C++ codebase at work and add a feature in, you think about what the memory is doing. Even without the borrow checker holding a knife to your throat, you consider the memory implications of the operations you perform, and tend to be more aware of potential bugs that could be introduced. You are just more _aware_ of what your code is doing, simply because of being burnt innumerable times from a poorly thought-out Rust snippet in your daily work.

The same applies to general software design. Rust forces you to think long and hard about how you want to achieve your goal, because what seems simple on the surface (look, a linked list!) can turn into a [labyrinth of possibilities](https://rust-unofficial.github.io/too-many-lists/) in an instant. You simply don't have the option of making a poor design decision and slapping some Flex Tape on when it needs to scale. By virtue of this, when you end up having to design a system for a different language, you still put in the same amount of thinking and try to do things right the first time around. This enables you to build fundamentally better software systems irrespective of the language being used.

# The art or the artist?

We can ask, what causes good software design? Is it the philosophy of the programming language, or the engineer using it? To that, I say it is both. No language can stop a bad engineer from building terrible software. Similarly, no language can stop a great engineer from building stellar software. But it can make it infinitely easier by eliminating the option of taking a shortcut, and that is what Rust does. Trivialising the choice of programming language for a problem by considering it just as a means to an end is short-sighted, and overlooks the larger impact of that decision. Languages are tools, but not merely so - they are tools that can be weaponised. The person wielding it is as much responsible for the outcome as the weapon itself. At the same time, writing off an engineer for their lack of expertise in a programming language is equally myopic. If you can win a battle with a katana, it won't take you long to learn to do it with a broadsword. In the end, what matters is **how you think**.
