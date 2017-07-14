---
layout: post
title: Move Semantic
---

In this document i will be providing an explanation for C++ move and i will assume you are a C++ programmer.

## Variables and addresses

```c++
int x = 0;
```

This line of code just declares a variable on the stack. imagine that is line is inside some sort of a function it will be mapped to the following

```assembly
mov     DWORD PTR [rbp-4], 0
```

This is just a mov instruction that does a simple thing it pushes the stack down by `sizeof(int)` and moves 0 to this memory location now from this we can conclude the following

- Variable x has a location in memory thus has an address that we can point to and put data into
- Constant 0 theoretically has no address ... technically it has since the compiled binary is loaded into memory and you can have a pointer and manipulate it but most OSs, Compilers, and everyone will [frown upon/prevent] you doing that

## Pointers and references

```c++
int x = 0;
int *z = &x;
int &y = x;
```

This block of code simply does one thing but it does it in two ways that's exactly the same.

- The first is we are making another variable `z` that's the size of `sizeof(int*)` which if you run in a 64-bit environment will be 8 and storing the address of x into it. in case we want to use it we simply deference it  `*z` and we get our x back
- The second is we are aliasing variable `x` or making a reference `y` to x which is under the hood a simple old pointer and the above two lines will generate the same exact assembly

```assembly
; int x= 0
mov     DWORD PTR [rbp-20], 0
; int *z = &x
lea     rax, [rbp-20] ;LEA is Load Effective Address instruction it gets the address of a variable
mov     QWORD PTR [rbp-8], rax
; int &y = x
lea     rax, [rbp-20]
mov     QWORD PTR [rbp-16], rax
```

So What's the use of references ?!

The answer is simple:

1. Syntactic sugar. it's easier and visually appealing to act as if this is a normal variable but under the hood you didn't copy anything at all especially function calls i.e. passing a variable by reference to a function
2. Compiler guarantees. for example compiler can guarantee that if you pass a variable by reference, this variable will always have an address and won't be `nullptr` . in addition to this once compiler notices a reference it can do a little more optimization like in-lining function calls ... etc.

## Function calls and convenience

Now if i were to make a function let's say `push_back` in `std::vector` implementation i would need this function to be easily called as following:

```c++
int x = 20;
vector.push_back(10);
vector.push_back(x);
```

Notice that i want this function to accept r-values and l-values, and also be efficient so it doesn't perform unnecessary copies now i have 3 options

```c++
void push_back(int x);
```

This option is not efficient since it passes x by value and with big values it will be a problem

```c++
void push_back(int& x);
```

This option will not accept r-values because they have no address

```c++
void push_back(int* x);
```

Once again, r-values has no addresses so this won't work you cannot write `push_back(&20);`

## Const solution

Well, to solve the above problem i as language Designer decided to give you a reference to the r-value but you have to promise me that you will not modify this variable by any means so i will allow this form of type to accept r-values

```c++
void push_back(const int& x);
```

Beside this i will allow you to write code that will be executed when a value being copied to override the default implementation

**Note:** C++ sets the assumption that in order to copy a value you can just copy it's bytes as a binary data so if i need to copy a type i just copy `sizeof(type)` #bytes to another location in memory nothing special here

So now you have two new toys to play with

1. Copy Constructors

```c++
Type(const Type& other)
{
  //copy data here
}

//can be called like this
Type x(y);
```

2. Copy assignment operator

```c++
Type&
operator=(const Type& other)
{
  //copy data here
}

//can be called like this
Type x;
//... do stuff
x = y;
```

Until now everything is fine. Enters C++11.

C++11 has a feature that triggered the addition of move semantics

## Smart pointers

Specifically `unique_ptr` let's implement it

```c++
template<typename T>
struct unique_ptr
{
	T* _ptr;

	unique_ptr()
		:_ptr(nullptr)
	{}

	unique_ptr(T* ptr)
		:_ptr(ptr)
	{}

	~unique_ptr()
	{
		if(_ptr)
			delete _ptr;
	}

	unique_ptr(const unique_ptr<T>& other)
	{
		//now what to write here ... !!!
	}

	unqiue_ptr<T>&
	operator=(const unqiue_ptr<T>& other)
	{
		//now what to write here ... !!!
	}
};
```

As you start implementing it you will face a problem how are you supposed to copy the unique_ptr? or if you supposed to copy it at all!
But if you don't copy it then how are you going to pass it around?!!

The unique_ptr would be possible if we could write this copy constructor

```c++
unique_ptr(const unique_ptr<T>& other)
	:_ptr(other.ptr)
{
	other._ptr = nullptr; //but we can't since other is a const reference
}
```

But we can't because other is passed as a `const reference` so to solve this we need the following new thing:

1. We need to be able to pass the value around efficiently
2. We need it to also accept r-values
3. We need to be able to edit it

## Universal References

Here enters universal references `&&` they accept r-values, pass value around efficiently, and they're not `const` so you can edit them

`&&` indicates that you are moving the value thus you don't care about it so i can write the following move constructor in our unique_ptr

```c++
unique_ptr(const unique_ptr<T>& other) = delete;

unqiue_ptr<T>&
operator=(const unqiue_ptr<T>& other) = delete;

unique_ptr(unqiue_ptr<T>&& other)
	:_ptr(other._ptr)
{
	other._ptr = nullptr;
}

unique_ptr<T>&
operator=(unique_ptr<T>&& other)
{
	if(_ptr)
		delete _ptr;
	_ptr = other._ptr;
	other._ptr = nullptr;
}
```

Here i have deleted the copy semantic, also i have introduced move semantic that **steals** the pointer from the other pointer and sets its internal `_ptr` to `nullptr`

Notice that i just copy the `unique_ptr`:

- It's no longer unique
- I end up with two variables pointing to the same memory, when one goes out of scope it will delete the memory and i will end up with a pointer to an invalid memory location furthermore if the second pointer attempts to destruct it will delete the pointer for the second time resulting an inevitable crash

So move semantic is simply stealing the data then reseting it to a default state of some sort like `nullptr` for pointers ... etc.

## std::move

So now i can write this code

```c++
#include <iostream>
void
func(int&& x)
{
	std::cout << "universal ref" << std::endl;
}

void
func(const int& x)
{
	std::cout << "const ref" << std::endl;
}

int main()
{
    int x = 10;
    func(x);
    func(10);
    //func(x); i want to pass x to the && func instead of const & how would i do that
    return 0;
}
```

Output:

```
const ref
universal ref
```

The problem here is that in the act of passing a variable to `func` and since it's in the stack the compiler will pass it to the `const&` function each time because the compiler thinks that since `x` is on the stack then we as programmers care about it. so the compiler needs to save its value for us. and passing it by && may result the value of `x` being changed to maybe a non-valid value like `nullptr` and i we use it later it will crash because of `nullptr` dereference ... of course x is an int here but imagine a complicated type.



Now if i want to call the `&&`  function i can do the following

```c++
#include <iostream>

void
func(int&& x)
{
	std::cout << "universal ref" << std::endl;
}

void
func(const int& x)
{
	std::cout << "const ref" << std::endl;
}

int main()
{
    int x = 10;
    func(x);
    func(10);
    //func(x); i want to pass x to the && func instead of const & how would i do that
    func(static_cast<int&&>(x));
    return 0;
}

```

Output:

```
const ref
universal ref
universal ref
```

A simple `static_cast` tells the compiler ... hey listen i know what i'm doing just convert this x to `int&&` and move it

Now let's wrap the cast into a function for convenience and give it an appropriate name to express our intent

```c++
template<typename T>
struct remove_reference
{ typedef T type; };

template<typename T>
struct remove_reference<T&>
{ typedef T type; };

template<typename T>
struct remove_reference<T&&>
{ typedef T type; };

template<typename T>
constexpr typename remove_reference<T>::type&&
move(T&& value)
{
	return static_cast<typename remove_reference<T>::type&&>(value);
}
```

Here we just provided this simple utility that's called `remove_reference` this simply takes a type `T` value, `T&` reference, `T&&` universal reference and gives you back the type `T` ... just ignore this if you aren't aware of C++ template meta programming

So the move function consists of a single line and it's the same `static_cast` that we have written before ... QED

**Note**: if you are aware of C++ templates stuff here are the collapsing rules for references

> Now, we have two kinds of references, where && means "I don't care about what may happen to the object", and & meaning "I may care about what may happen to the object, so you better watch what you're doing". With this in mind, the collapsing rules flow naturally: C++ should collapse referecnces to && only if no one cares about what happens to the object:
>
> & & -> &
> & && -> &
> && & -> &
> && && -> &&

Simply think of the move function it's a generic function that accepts any type `T` if i pass to it `T` by reference the type would look like this `move(T& && value)` and it will resolve to `move(T&)` ... etc.

Finally this code

```c++
#include <iostream>
#include <utility>
void
func(int&& x)
{
	std::cout << "universal ref" << std::endl;
}

void
func(const int& x)
{
	std::cout << "const ref" << std::endl;
}

int main()
{
    int x = 10;
    func(x);
    func(10);
    //func(x); i want to pass x to the && func instead of const & how would i do that
    func(static_cast<int&&>(x));
    func(std::move(x));
    return 0;
}

```

Output:

```
const ref
universal ref
universal ref
universal ref
```

