---
date: "2018-03-21T00:00:00Z"
aliases:
  - /2018/03/21/hlsl.html
image: /assets/images/posts/002/LanguageAreas.png
title: HLSL
---

## HLSL?

Hlsl is the language the "juicy" parts of unity shaders are written in. The parts that contain custom logic and eventually decide what is drawn where on screen. It's the language Microsoft designed to work with their Direct3D API to write gpu programs. Strictly speaking most Unity shaders are tagged as being written in CG which is short for `C for Graphics`, but CG shares most of it's syntax and features with hlsl and was deprecated in 2012, so wrongly referring to it as hlsl leads to you being able to find better results in search engines and not much else. In theory Unity also supports writing glsl shaders, the language designed for OpenGL, but since there are more examples for hlsl and in the end the languages will be converted into whatever language the platform we're exporting to needs anyways we're gonna stick to hlsl.

For learning shaders in Unity I recommend you do not start your programming jouney here. Shaders are tricky to debug, are pretty limited in the things they can do and in many cases you have to thing in a slightly different direction than in most programming contexts. So I'm going to assume you know `types`, `variables`, `classes`, `methods`, `loops`, `if statements` and similar basics.

## Builtin Types

Heres some of the builtin types we're going to use when making shaders.

### Scalar Values

The different number types in unity hlsl are `fixed`, `half` and `float` for the floating point numbers and `int` as well as `uint` for integer numbers.

On mobile GPUs fixed is a number between -2 and 2 with 1/256th precision, half is a 16bit floating point number and float a 32bit floating point number. On Desktop GPUs all of those values are 32bit floating point numbers, so you often see me use float everywhere out of convenience, but its a nice tool to have for optimisation later.

Integers are numbers that can only have whole values, int can hold positive and negative values, uint only negative ones, which can have a slight performance benefit.

In addition there is also the simple datatype bool which simple holds the value `true` or `false` so we can store the result of checks in them. If you use boolean values with other numbers (like adding or multiplying them) they behave like the number 0 in case its `false` or 1 if its `true`.

### Vector Values

On top of the one dimensional types we can also access several vector types with up to 4 values by just adding a number to the end of a scalar(simple, one dimensional) type like `fixed4`, `float2` or `half3`. We use those types for example to save texture coordinates, colors and positions.

When accessing one of the values inside a vector we use in order either `vector.x`,`vector.y`,`vector.z`, and `vector.w` in the context of coordinates or `vector.r`,`vector.g`,`vector.b`, and `vector.a` in the context of colors. Alternatively we can also access values like this: `vector[2]` (this is 0 based so the 4 values of a 4d vector would be 0, 1, 2, and 3).

In case we want to get multiple values from a vector and put them in another vector, we don't have to construct that new vector ourselves, we can use swizzling. Swizzling means writing multiple vector subvalues after the dot. It looks like this:

- `vector.xy` - only get first 2 values of vector and put them in a 2d vector.
- `vector.zyx` - get first 3 values of vector and reverse their order.
- `vector.xxxx` - take the first value of a vector and construct a 4d vector full of that value.

### Matrix Values

Like vectors expand types in one direction, matrices expand them in 2 directions. The syntax for them is to append `number x number` to the sclar types. Like `float4x4`, `half3x2` or even `bool2x4`. Accessing the members of matrices works either via the square brackets like `matrix[3][2]` with the numbers first declaring the row, then the column. Or via accessors like `_m32`. Those accessors can also be used for swizzling which looks like this `matrix._m03_m13_m23` to get the 3 first values of the last column and write them into a 3d vector. If we use the square bracket version and only use one pair of brackets we get the vector that defined that row.

Luckily we nearly never have to access values in matrices, so thats more a tool to come back to than something you have to understand now.

### Textures

Hlsl also has types to describe textures. For the most part those aren't that exiting, but you can read their pixels with the tex2D(texture, coordinate) function...? We're gonna learn more about them soon, I promise.

## Math

In addition to the regular + - \* / operators for simple math and > < == != ! >= <= && and \|\| for comparisons, hlsl also gives us access to many math functions like `abs`, `dot`, `lerp`, `pow`, `min`, `atan2` as its intrinsic functions (you can look up all of them [here](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-intrinsic-functions)).

There are also a few shorthands like += \*= -= and /= which modify a variable and then assign to itself again as well as var++ and var-- which add or subtract 1 from a given variable.

Multiplications of a scalar type and a vector type return a vector type with the same amount of dimensions all of them multiplied by the number. so `float2(2, 7) * 3` is equal to `float2(6, 21)`.

For multiplications of matrices with vectors to transform the vector we use the [mul](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-mul) function. I this is a bit black magic at the beginning, so my best recommendation is to learn how to copy the cases where matrix multiplcation is used into your context.

## Custom Types

In addition to the builtin types we can also add our own types. The syntax for adding your own types is like this (the semicolon at the end is important!):

```glsl
struct typeName{
  float variable;
  float2 otherVariable;
};
```

In theory we can also use the `class` keyword and use inheritance, member functions amd even interfaces, but I've never encountered them in any shaders so I won't explain them here, if they come up, the time is then. In case you do want to use those features, the [syntax is like in C++/C#](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/overviews-direct3d-11-hlsl-dynamic-linking-class).

Like with vector types we access member variables of a custom type with a dot so for example `instance.variable` or `instance.otherVariable.x` if we go into members of members.

## Variables

All datatypes in hlsl are value types. That means as soon as we have a value we can change it, no need to create it via `new` or similar.

If we want to create vector types we can do that by calling the types like functions. In those cases the amount of subvalues in the argument types have to add up to the amount of values in the target type. so we can create a float4 by passing in `4 floats`, `2 float2`, a float, a float2 and another float or just one other float4. So we can create datatypes like this:

```glsl
typeName instance;
instance.variable = 3.14;
instance.otherVariable = float2(3, 1.4);
```

Variables we declare can either be inside of a function, in which case they can only be accessed by other parts of that function that come *later* than the variable declaration(like in most programming languages) or be outside of functions in which case they can be accessed from all functions in the shader, no matter the order (but its common to declare them at the top, above the function to easily find them).

## Functions

Most functions in hlsl are in the global scope. That means they're not part of any datatypes and we can call them from anywhere. They can take multiple (or none) arguments and return a value. If your function doesn't return a value, you have to declare the return type as `void`. A typical function syntax looks like this:

```glsl
returnType functionName(argType arg1, otherArgType arg2){
  //do some stuff and calculate returnValue

  return returnValue;
}
```

To call a function we just write the function name followed by brackets and the arguments in the backets in case we need any. If there are multiple functions with the same name but different argument types, hlsl will automatically find the one that fits the arguments we called it with.

## Control Flow

For many shaders it's enough to execute one command after the next without missing or repeating any of them. For for a lot of them it's also important that we choose between two paths of code. It's common knowledge that using control flow in your shaders is detremental and especially on mobile GPUs will affect your performance and that you should use functions like `step` instead and multiply a bunch. This is plain wrong since those functions also use branching inside of them and making yourself work with them likely makes your code more complex and harder to read. It's true that it can happen that your GPU has to calculate both sides of a branch and throw away one of them, but thats also the case when using different techniques, so don't abstain from if statements for ideological reasons and make your code prettier instead.

### if statements

If statements look like this, if the condition is true, the do thing block will be executed, otherwise the do other thing block, never both.

```glsl
if(condition){
  //do thing
} else {
  //do other thing
}
```

In this the `else` and its block are optional. The brackets around the condition are mandatory. If you don't use the curly braces the statement will only affect the next line(til the next semicolon, not til the next newline), not the lines after that. The condition can either be a boolean, a number (in which case 0 is false and any other value including negative ones are true) or a operation returning one of the two. A small beginner tip is that if you have a value that is the opposite of what you want it to be (false when you want to execute some code and true if you don't), just prefix it with a exclamation mark, that flips true values to false and false ones to true.

### Loops

The other way of controlling which code is executed and which isn't are loops. While loops are the simpler kind of loop. They are executed perpetually as long as the condition they're defined with isn't true anymore. If it isn't true at the beginning they're not executed at all. They look like this:

```glsl
while(condition){
  //do things
}
```

Its important that somewhere in the do things block you eventually change variables in a way that the condition isn't true anymore, otherwise the loop win run forever which is bad and can even crash your whole editor (you can luckily fix the shader without having the editor open so theres no chance of a lasting destruction I'm aware of).

Another kind of loop is the for loop which adds some syntactic sugar to iterate and count, they're defined like this.

```glsl
for(beforeLoopLogic; condition; inLoopLogic){
  //do things
}
```

Thats a lot more freedom that you usually need so the typical less confusing usecase where the variable `index` counts from `0` to `maxValue-1` is:

```glsl
for(uint index=0;index<maxValue;index++){
  //do things
}
```

this code is equivalent to this code which uses a while loop instead:

```glsl
uint index = 0;
while(index < maxValue){
  //do things

  index++;
}
```

Both loops also support the keywords `break` and `continue`.

Having a `break` statement in your code makes the code jump to the end of the next loop. This also allows for escaping while loops that would otherwise be infinite.

Having a `continue` statement makes the loop jump to the start of the next iteration of the loop instead.
