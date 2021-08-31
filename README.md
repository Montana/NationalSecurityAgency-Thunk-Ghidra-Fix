# NSA (National Security Agency) Ghidra Thunk Function fix by Montana Mendy

In the image below you can see obviously there are a lot of `thunk` functions but honestly to me they just look like `printf's`. I don't know how to fix this so I can actually get readable function names, not even sure if there is a way:

<img width="437" alt="Screen Shot 2021-08-31 at 5 33 00 AM" src="https://user-images.githubusercontent.com/20936398/131503326-15190cdc-e597-40c1-a942-0b889551f60a.png">

I'm aware apart from situations like `MSIL` or `Java`, where there is a lot of metadata present in the compiled binary, `Ghidra` has no way to determine function names automatically. A slightly worse statement is true for variables I'm assuming, given that in many architectures there isn't even a concept of a variable in machine code.

Looking at the `Ghidra` documentation:

```java
public interface ThunkFunction
extends Function
```
From what I've read on the the official `Ghidra` documentation,  and my understanding is, the `_ThunkFunction` corresponds to a fragment of code which simply passes control to a destination function. All Function behaviors are mapped through to the current destination `function._`

Reading this makes me think the API function system was already correctly named, and not incorrectly named, so `Ghidra` was calling something that didn't exist. This being said, when there is no parameter for a given argument, the argument is passed in such a way that the receiving function can obtain the value of the argument by invoking `va_arg`.

<img width="1140" alt="54684407-85ef6a80-4b14-11e9-8704-77ad3ce530d1" src="https://user-images.githubusercontent.com/20936398/131503398-6a71a5de-10c3-4add-9f69-fd2abc7a46ea.png">

In the case my hunch is right and it is `printf`, which has variadic arguments, will it be necessary to select `Varargs` on the right.

So I guess my question right now is - would it be best to pass an array of `primitives` as `Varargs`. So if I wanted to cast an existing `int[]` array, would it look like the following?

```java
int[] ints = new int[] { 1, 2 };

Integer[] castArray = new Integer[ints.length];
for (int i = 0; i < ints.length; i++) {
    castArray[i] = Integer.valueOf(ints[i]);
}

System.out.println(String.format("%s %s", castArray));
```

<img width="660" alt="Screen Shot 2021-08-31 at 5 11 13 AM" src="https://user-images.githubusercontent.com/20936398/131500380-e2bf43ad-7a01-47d8-a258-b62be4d993ce.png">

## Three solutions 

Interestingly, seems like I found a quick fix just as I posted. I nested some of the functions. Thus, I looked into `Function argument lists`. Which appeared exactly what I was looking for.

The pack expansion may appear inside the parentheses of a function call operator, in which case the largest expression or `braced-init-list` to the left of the ellipsis is the pattern that is expanded. A quick example of this is the following:

```cpp
f(&args...); // expands to f(&E1, &E2, &E3)
f(n, ++args...); // expands to f(n, ++E1, ++E2, ++E3);
f(++args..., n); // expands to f(++E1, ++E2, ++E3, n);
f(const_cast<const Args*>(&args)...);
// f(const_cast<const E1*>(&X1), const_cast<const E2*>(&X2), const_cast<const E3*>(&X3))
f(h(args...) + args...); // expands to 
// f(h(E1,E2,E3) + E1, h(E1,E2,E3) + E2, h(E1,E2,E3) + E3)
```
_This was formally, the expression-list in a function call expression is classified as initializer-list, and the pattern is the initializer-clause, which is either an assignment-expression or a braced-init-list_. 

Secondly, I quickly looked at parenthesized initializers, which seemed to work as well. This can mostly be explained via, a pack expansion may appear (more often than not) inside the parentheses of a direct initializer, a function-style cast, and other contexts (member initializer, new-expression, etc.) in which case the rules are identical to the rules for a function call expression above, so in essence a recursive expression:

```cpp
Class c1(&args...);             // calls Class::Class(&E1, &E2, &E3)
Class c2 = Class(n, ++args...); // calls Class::Class(n, ++E1, ++E2, ++E3);
::new((void *)p) U(std::forward<Args>(args)...) // std::allocator::allocate
```

The last fix I was looking at was using `Lambda captures`, which looks like this: 

```cpp
template<class ...Args>
void f(Args... args) {
    auto lm = [&, args...] { return g(args...); };
    lm();
}
```
_Reason for this is, is a parameter pack may appear in the capture clause of a lambda expression._

## Top down explanation

The nested lambda expressions and invocations below will output `123234`:

```cpp
int a = 1, b = 1, c = 1;
auto m1 = [a, &b, &c]() mutable {
  auto m2 = [a, b, &c]() mutable {
    std::cout << a << b << c;
    a = 4; b = 4; c = 4;
  };
  a = 3; b = 3; c = 3;
  m2();
};
a = 2; b = 2; c = 2;
m1();
std::cout << a << b << c;
```

### Conclusion

* When you capture a reference by value, you make a copy of the object that it points to.

* When you capture a reference by reference, you make a reference to the object that it points to.

* Here, "capturing a reference" applies equally well to capturing actual references, and to capturing reference-captures of enclosing lambdas.



