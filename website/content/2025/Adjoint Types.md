+++
# The title of your blogpost. No sub-titles are allowed, nor are line-breaks.
title = "Adjoint Types"
# Date must be written in YYYY-MM-DD format. This should be updated right before the final PR is made.
date = 2021-08-13

[taxonomies]
# Keep any areas that apply, removing ones that don't. Do not add new areas!
areas = ["Programming Languages"]
# Tags can be set to a collection of a few keywords specific to your blogpost.
# Consider these similar to keywords specified for a research paper.
tags = ["type systems", "substructural logic", "functional programming"]

[extra]
author = {name = "Sophia Roshal", url = "https://sophiasr.github.io/" }
# The committee specification is  a list of objects similar to the author.
committee = [
    {name = "Jan Hoffmann", url = "Committee Member 1's page"},
    {name = "Marijn Huele", url = "Committee Member 2's page"},
    {name = "Aditi Kabra", url = "Committee Member 3's page"}
]
+++


Functional programming languages are growing in popularity due to their strong guarantees and ease of proving programs correct. However, their memory usage is difficult to optimize. This blog post presents a type system and functional programming language that is a step towards enabling memory-reuse optimizations in compilers for functional programming languages. To do this, we will use Adjoint Types. At a very high level, Adjoint Types let programs mix types with (possibly) different properties together, where some types may be amenable to certain optimizations (but may be more difficult for programmers to reason about) while other types may not be as amenable to optimizations (but are more straightforward to program with). For those who are interested in formal definitions, there are paragraphs colored blue with formalizations. However, if you just want to build your intuition or read about how you might write code in such a language, feel free to skip any such blue sections.


# What is a type system?
So we've said some words but what do they actually mean? We'll start small and build up to the whole "Adjoint Types" thing by the end. If you know what a type system is, feel free to skip this section and move on to the fun stuff!

At a high level, a type system is a method for catching (some) incorrect programs before your code executes. Consider, for example, a function to square natural numbers.



``` 
defn square (x : nat) : nat = 
        x * x
```

If we call it on a natural number as follows:

``` 
square 42
```

We would get the output: 

```
1764
```

What happens if, instead of giving it a number, we give it a string?

```
square "the answer to the ultimate question of life, the universe, and everything"
```
 
One possibility is we don't notice this mistake until we run our code, and then we get a runtime error telling us we did something wrong. This is what would happen in a language such as python with no static type checker. Static type systems allow this error to be caught before runtime. We can specify rules that all programs must follow and statically (without running the code) determine whether the code follows them. For example, we could define a rule for functions that says that if we know the function should take in type A and output type B, then we can only apply it to something of type A (and we will then be guaranteed that, if the function terminates, the output will have type B). These rules are formalized via inference rules, demonstrated below.  To read the rest of this blogpost, you will not need to be able to read *inference rules*, so feel free to skip this next blue bit.


<span style="color:blue"> \\[ \frac{\Gamma \vdash f : A \rightarrow B\ \ \ \ \Gamma \vdash e : A}{\Gamma \vdash f\ e : B} \\] This is just a formal way to write what was described before. This says, that, to show that \\(f\\) applied to \\(e\\) has type \\(B\\), we must show that \\(f\\) is a function which takes in something of (some) type \\(A\\) and outputs something of type \\(B\\), and we must also show that \\(e\\) has type \\(A\\). The \\(\Gamma\\) represents what is called a *context* which keeps track of all variables in scope and what their types are. We can provide similar typing rules for other constructions, not just functions.
 </span>

With our squaring example, this means that since the ``square`` function takes in a number and outputs a number, the input we actually give the function when we call it must also be a number, and a type error will be raised if this is not the case. We can do these sorts of checks without actually running the code, because at this stage we don't care what the value of any term is, just whether it has the right *type*. Most languages in use today have some kind of static type system including languages such as Java, Haskell, OCaml, C, and many others. Python, on the other hand, does not.

# Substructural Types
SSo now we have some understanding of what a type system does and how we can formally define one. Substructural types extend the usual type systems with a way to track how exactly variables in the program are used. This gives rise to some guarantees about which compiler optimizations are safe to perform.

We now think of variables as resources that we can consume or not.

For example, if we wanted to come up with a way to express baking oatmeal cookies formally, we could have something like

> 3/4c oil, 1c sugar, 1 egg, 1/4c water, 1tsp vanilla, 1c flour, 1/2 tsp baking powder, 3c oats \\(\rightarrow\\) oatmeal cookies
>
<p></p>
Once we've used all the ingredients to make cookies, we should not be able to use them again. However, in a standard type system, once you have any oil, you have unlimited oil. Instead, we turn to substructural types, which use the structural properties called weakening and contraction. Weakening means that a resource doesn't have to be used. Contraction means that a resource can be used more than once. Unless you're a fully automated bakery, you're probably not programming cookie baking very often, so why is substructurality a feature we want?

One important use case is for memory management. Linear types that cannot be weakened or contracted guarantee that every resource is used exactly once. This is useful because

1. Since we know that a resource is used at most once, once it is used we can free the memory used to store it and potentially immediately store new data there

2. Since we know that everything is used at least once, we know that there won't be any resources still left stored in memory at the end of execution. This means we can guarantee our program is free of memory leaks without needing a garbage collector
  
But as most people who have written some code in their lives know, we don't always use all the variables we have defined, and sometimes we may want to use a variable more than once.

Consider the following piece of code which takes the boolean ``or`` of two boolean inputs. In this language, functions are lazy, and so we will not read inputs if they are not used. The standard implementation short circuits if the first input is true, never reading the second input.

```
defn or (b1 : bool) (b2 : bool) : bool = 
    match b1 with 
    | True => True
    | False => b2
```

This is not valid code in a purely linear system, instead we would have to also read the second input, and so we would need to rewrite the code. There are several ways to do so, here, we choose to explicitly read the input, and ignore its value:

```
defn or (b1 : bool) (b2 : bool) : bool = 
    match b1 with 
    | True => match b2 with 
            | True => True 
            | False => True 
    | False => b2
```
As a demonstration of wanting to use variables more than once, consider the ``square`` function from earlier, copied here again for reference:

 ```
 defn square (x : nat) : nat = 
    x  *  x
```
We use the variable \\( x \\) twice here, and so this code snippet is not valid in a purely linear type system. Fixing this is even trickier than the previous example, and depends on the representation of the type ``nat``. Let's assume that natural numbers are internally represented as unary numbers: they are either zero, or the successor of another natural number. We will see how to define such a type in the next section, but for now, we assume this has already been defined. Now, we can write a ``copy`` function on such numbers. This function takes in a natural number and produces a pair of natural numbers, which should both be equal to the input.

 ```
 defn copy (n : nat) : (nat * nat) = 
    match n with 
    | zero => (zero,zero)
    | succ n' => match copy n' with
                | (n1,n2) => (succ n1, succ n2)
 ```

If the number is zero, then we return two copies of zero. If the number is the successor of another number, then we recursively call ``copy`` on that other number, break down the components of the pair, and then rebuild the pair, taking the successor of each component. With this now implemented, we can return to taking the square:

 ```
 defn square (x : nat) : nat = 
    y = copy x
    match y with 
    | (y1,y2) => y1 * y2
```
We were able to do this because of how we defined ``nat``, but it is not always possible to define such copy functions for all types. It is also impossible to define a general copy function and must be defined for each type. Similarly, in the first example, we were able to just read the second input and ignore its value, but this is again not always possible to do. This is a large amount of overhead just to be able to (sometimes) reuse or drop data we have defined.

So what do we do if we want to exploit the benefits of linear and other substructural types while also allowing the user to write code without thinking too hard about ensuring that everything is used a specific number of times?

# Adjoint Types
It's possible to allow certain pieces of your program to be just linear, and others to be just unrestricted. However, this is insufficient, as it means that the linear and non-linear parts of our program cannot interact with each other, since purely linear type systems cannot consider unrestricted pieces of code and vice versa. As an example of where we want some interaction, we may want to define a binary tree that itself is linear, but whose elements are unrestricted so that multiple comparisons can be done on each individual element. To support this sort of interaction, we can use Adjoint Types.

Adjoint Types let us mix types that have different structural properties and let them interact with each other. This is accomplished via the concept of modes. We can think of a mode as an annotation on types that specifies which structural properties that type will have. We say that a mode admits weakening if, when a type is annotated with it, terms of that type can go unused. Similarly, we say that a mode admits contraction if, when a type is annotated with it, terms of that type can be used more than once.

Rather than writing just \\( A\\) for a type, we now write \\( A_m \\) where \\( m \\) is the mode of the type. These modes are not symmetric in how they interact with each other. For example, it should not be possible to use a linear resource to construct an unrestricted one directly. If this were the case, we would be able to arbitrarily copy linear resources. This property is called independence.

To provide interaction between modes, we give constructors that can take a term at one mode and wrap it in a term at another mode. We do so via two "shifts": *down* written \\( \downarrow^n_m A_n \\) which wraps something at a more permissive mode in something at a possibly less permissive mode and *up* \\( \uparrow^m_k A_k \\) which wraps something at a less permissive mode in something at a possibly less restrictive mode. Down is the type of a term written as \\(\langle e \rangle\\) while up is the type of a term written as \\(\mathbf{susp}\ e\\). To then re-extract the inner term from these shift, we write \\(\mathbf{match}\ e\ \mathbf{with} \mid \langle x \rangle => e' \\) for down, and \\(e.\mathbf{force}\\) for up.

The formalization with typing rules for these new constructors can be found below in blue, but for the audience who prefers less Greek and more code, skip to the next section to see how you might write code in a language with an Adjoint Type system.


<span style="color:blue"> \\[ \frac{\Gamma' \geq n\ \ \ \ \Gamma' \vdash e : A_n}{\Gamma_W;\Gamma' \vdash \langle e \rangle : \downarrow^n_m A_n} \ \ \ \ \ \ \ \ \ \  \frac{\Gamma \vdash e : A_k}{\Gamma \vdash \mathsf{susp}\ e  : \uparrow^m_k A_k}\\] 
\\[ \frac{\Gamma \vdash e : \downarrow^n_m A_n \ \ \ \ \Gamma' \geq m \geq r \ \ \ \ \Gamma', x:A_n \vdash e' : C_r }{\Gamma;\Gamma' \vdash \mathsf{match}\ e\ \mathsf{with}\ (\langle x \rangle \Rightarrow e') : C_r} \ \ \ \ \ \ \ \ \ \  \frac{\Gamma' \geq m \ \ \ \ \Gamma \vdash e : \uparrow^m_k A_k}{\Gamma_W;\Gamma' \vdash e.\mathsf{force}  : A_k}\\]
There are several things going on here. First, we read these rules a bit differently than the function rule that we previously described. We think of these rules almost in a backwards direction. Rather than thinking the top implies the bottom, we think, we are trying to prove the bottom, what needs to be proven to get to that conclusion? With this in mind,we can *presuppose* independence of the conclusion, that is, if the conclusion is \\(\Gamma \vdash A_m\\) we can assume that everything in \\(\Gamma\\) is at a mode that is at least as permissive as the mode \\(m\\). We may however need to ensure this property in the premises, in which case we write \\(\Gamma \geq m\\) to indicate that we are checking that everything in \\(\Gamma\\) is at a mode at least as permissive as \\(m\\).</span>

<span style = "color:blue">Next, we also presuppose that the shifts are *well-formed*. That is, in the down shift \\(\downarrow^n_m A_n\\) we assume that \\(n\\) is at least as permissive as \\(m\\) and in the down shift \\(\uparrow^m_k A_k\\) we assume that \\(m\\) is at least as permissive as \\(k\\). </span>

<span style = "color:blue"> Lastly, we write \\(\Gamma_W\\) to indicate that everything in \\(\Gamma\\) is weakenable (and thus can be dropped).
 </span>

 <span style = "color:blue"> Now, we can actually read these rules. In the first rule, we presuppose 
 \\(\Gamma_W;\Gamma \geq m\\), and \\(n \geq m\\), however that does not give us the independence property 
 needed for \\(\Gamma \vdash e : A_n\\), and so we must explicitly check \\(\Gamma \geq n\\). We do not require this property for anything that isn't used to derive \\(e : A_n\\) as long as we can drop it (it admits weakening). In the second rule, we presuppose \\(\Gamma \geq m\\) and \\(m \geq k\\) and so by transitivity we have \\(\Gamma \geq k\\) so we have the independence property we need and no further checks are required. Similar reasoning explains the checks needed in the other rules. </span> 

# The Language
We will review some code examples, some of which type check and some of which do not, to understand how different modes can interact with each other.
 
 ## Defining Modes

We define modes via the keyword *mode* followed by the name we want to give it, a keyword that states what structural properties the mode has, and any relationship we want this mode to have with other declared modes. All our examples will consider only the following two modes, however, it is possible to define others as well:


```
mode L linear
mode U structural :> L 
```

This states that the mode \\(L\\) is linear (admits neither weakening nor contraction), and the mode \\(U\\) is unrestricted (admits both weakening and contraction). The \\(U :> L\\) establishes the relationship that data at mode \\(U\\) cannot rely on data at mode \\(L\\), but data at mode \\(L\\) can rely on data at mode \\(U\\) and that \\(U\\) must be at least as permissive as \\(L\\).


## Defining Types
We define types via the keyword *type* followed by the name of the type and a mode annotation.

To start with, we define unary natural numbers.

```
type nat[m] = +{'zero : 1, 'succ : nat[m]}
```

The annotation \\([m]\\) says that the type ``nat`` has the mode \\(m\\), the + syntax denotes a sum type. That is, nat can be either zero, which has the unit type, or the successor of another natural number.

Next, let’s define lists of unary numbers. This gets a bit more interesting as we start seeing the interaction with modes.

```
type list[m k] = +{'nil:1, 'cons: <nat[k]> * list[m k]}
```
The annotation \\([m\ k]\\) says that the type ``nat`` has the mode \\(m\\) and the mode \\(k\\) appears in the definition as well.

Looking at the type definition itself, we see that a list can either be empty (represented by the tag ``'nil``) or ``'cons`` of an element and the rest of the list. The interesting thing here is the representation of the elements. While the mode of all the components of the ``'cons`` need to be at the same mode, we can still have the elements of the list appear at a different mode by wrapping the elements in a (down) shift. The nats themselves can be at the mode \\(k\\) while allowing the full list to still be at the mode \\(m\\).

We can then construct some lists. First, a few legal constructions:


```
(* A list with the element 1 in which the element is linear as is the list itself *)
decl l1 : list[L L] 
defn l1 = 'cons (<'succ ('zero ())>,'nil ())

(* A list with the element 1 in which the element is unrestricted but the list is linear *)
decl l2 : list[L U] 
defn l2 = 'cons (<'succ ('zero ())>,'nil ())

(* A list with the element 1 in which the element is unrestricted as is the list itself *)
decl l3 : list[U U]
defn l3 = 'cons (<'succ ('zero ())>,'nil ())
```

These are all ok because the shift takes something of a more or equally permissive mode, and wraps it in a less permissive mode, which is the requirement for the down shift. The following definition does not respect that and thus would not be allowed:

```
(* Trying to construct a list which itself is unrestricted but has linear elements *)
decl lbad : list [U L]
defn lbad = 'cons (<'succ ('zero ())>,'nil ()) 
```

If we think about this intuitively, it should make sense that this construction is not allowed. If a list itself is unrestricted (can be read multiple times) but the elements are linear (can be read exactly once), we run into a problem. To read the list, we need to read the elements, and if the elements have already been read, that becomes impossible.


## Writing Some Code
Now, let’s write some code that uses the ``list`` type. What if we want to append two lists?

```
defn append l1 l2 = 
    match l1 with 
    |'nil () => l2
    |'cons (<hd>,tl) => 
        'cons (<hd>,append tl l2)
```

Looking at all the branches in the definition, we use all our resources exactly once, and so we should be able to give this definition all of the below type declarations.

```
decl append (l1 : list[L L]) (l2 : list[L L]) : list[L L]
decl append (l1 : list[L U]) (l2 : list[L U]) : list[L U]
decl append (l1 : list[U U]) (l2 : list[U U]) : list[U U]
```

There is one more slightly more subtle declaration as well. Because we deconstruct the shift around the head element and reconstruct it, it could be reconstructed at a different mode, and so we can also have the following type declaration:

```
decl append (l1 : list[U U]) (l2 : list[L U]) : list[L U]
```

Other declarations such as

```
decl append (l1 : list[L L]) (l2 : list[U U]) : (list[U U])
```
would not be allowed as they violate the independence property. Again, thinking about this intuitively, if we take a list with unrestricted elements and append it to a list with linear elements, the resulting list should not have all unrestricted elements as that would convert the linear elements of the first list into unrestricted ones.

Now, let's look at some use cases of the up shift. Another common list operation is ``map``, which applies some function to each element of a list, producing a new list. Let's first write it how we would normally write it without using any up shifts. We will start by giving it a general form for a type declaration (without really thinking about what the modes might actually be, just using place holder variables for now).

```
decl map (f : nat[k] -> nat[k]) (l : list[m k]) : list[m k]
```

Then, writing the usual map code:

```
defn map f l = 
    match l with 
    |'nil () => 'nil ()
    |'cons (<hd>,tl) => 'cons (<f hd>, map f tl)
```
If we take a look at the first branch, we see that ``f`` is not used at all. This means that the mode of ``f`` (in this case, \\(k\\)) must admit *weakening*.

Looking at the second branch, we see that ``f`` is used twice (first in the application to the head element, and then when it is passed to the recursive call to map). This means that the mode of ``f`` must admit *contraction*. Knowing this, and knowing that the mode of ``f`` is the same as the mode of the elements of the list, we would need to conclude that the mode of the elements of the list themselves must also admit both weakening and contraction. Only the following type declarations would then be allowed:

```
decl map (f : nat[U] -> nat[U]) (l : list[L U]) : list[L U]
decl map (f : nat[U] -> nat[U]) (l : list[U U]) : list[U U]
```

This is, however, overly restrictive. The elements themselves are only read once throughout this definition, and so should not have this requirement. To address that, we can wrap the function ``f`` in an *up shift*. Again, we start by giving the general form of the type:


```
decl map2 (f : [k'] (up[k] (nat[k] -> nat[k]))) (l : list[m k]) : list[m k]
```

The syntax here means that we wrap the function type \\(\mathsf{nat[k] \rightarrow nat[k]}\\) in an up shift as such \\(\mathsf{\uparrow_k^{k'} nat[k]\rightarrow nat[k]}\\). Now, we can write the new implementation with this type in mind:

```
defn map2 f l = 
    match l with 
    |'nil () => 'nil ()
    |'cons (<hd>,tl) => 'cons (<f.force hd>, map2 f tl)
```

Now, only the mode of the term ``f`` will be required to admit weakening and contraction, and it is no longer the same as the mode of the elements of the list. the mode of ``f`` is \\(k'\\) not \\(k\\), and so only \\(k'\\) will be required to admit weakening and contraction. When the function actually needs to be applied, the up shift wrapper is eliminated via \\(\mathsf{f.force}\\) leaving just the function. Now, map can have more valid type declarations:

```
(* translations of previous *)
decl map2 (f : [U] (up[U] (nat[U] -> nat[U]))) (l : list[L U]) : list[L U]
decl map2 (f : [U] (up[U] (nat[U] -> nat[U]))) (l : list[U U]) : list[U U]
(* new *)
decl map2 (f : [U] (up[L] (nat[L] -> nat[L]))) (l : list[L L]) : list[L L] 
```
# Conclusion
This blog post gives a high level summary of an Adjoint Type system and how one may write code in it. It does not discuss how exactly the compiler goes about exploiting the substructural properties of the system for optimizations. If you would like to explore this programming language and its implementation, it is available here: [SNAX](https://bitbucket.org/fpfenning/snax/src/main/). The examples from this blog post can be found under the [play/Sophia/BlogPost.adj](https://bitbucket.org/fpfenning/snax/src/main/play/Sophia/BlogPost.adj) file. For technical reasons related to memory layout, these examples are slightly different: all recursive types are guarded with a down shift. We ommitted this detail here for readability. Many more examples can be found throughout the directory. Some relevant papers and preprints can also be found here: [Adjoint Natural Deduction](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.FSCD.2024.15) and here: [Mode Inference for Adjoint Types](https://drive.google.com/file/d/1v156zeVt0nqoxtNhuRbi7JVFbD4xQJ2P/view?usp=sharing). The first of these discusses the type system as overviewed in this blog post, and is where you can find the formal type system. The second discusses an extension to that system that permits mode inference, not just checking.

PS: The oatmeal cookies recipe is a real recipe :) preheat oven to 350 degrees, mix all the ingredients (the egg should be beaten first), place dollops of dough on a lightly oiled baking sheet, spaced out a bit as they can spread, and bake for 12-15 minutes. You can add chocolate chips, nuts, dried fruit, or any other mixins your heart desires.
