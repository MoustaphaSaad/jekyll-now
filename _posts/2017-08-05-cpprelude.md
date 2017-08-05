# CPPrelude

Hello,

Two months ago I was frustrated by Microsoft's implmentation of the C++ STL mainly because in debug mode it's unusable. Try it yourself if your program uses STL containers heavily your debug mode will be slow as hell. you won't be able to use it.

I encountered this problem two times in real world. and those two instances were not just me caring about performance but I literally was not able to use my program in debug mode nor i was able to perform simple debugging simply because a step of my program was taking about 5 minutes to execute. and the second time was at work we were not able to debug a certain area of the program because it took about 20 minutes to execute. this problem is not helped by the fact that MSVC compiler is terrible at optimizations even in release. I tried to compile same code using MSVC/clang/g++ all running on the same machine and OS and every time MSVC was the worst.

So at the end of the day I teamed up with [**Nora Abdelhameed**](nora.abdelhameed@gmail.com) and we started implementing our own container library.

## Philosophy

What is a computer? a computer is simply an implementation of a Turing machine. A computer is simply a machine to run instruction and that's why we have a job called programmer who's responsibility is to provide the correct instruction to make this machine simulate other machines for example it can simulate a calculator, phone, music player, painter ... etc.

The de facto computer science degree -at least where I live- you go in you learn Java or C++/OOP way. you go out and work assuming this ideology because you simply aren't aware of other views. in fact many of my supposed "teachers" -these guys have PhDs!!- were not able to write code at all let alone writing good working code. my compilers teacher I can claim he didn't even write a compiler in his life. my Theory of Computation teacher were not aware of the significance of this subject at all. Heck I bet she didn't even know that what she was teaching were linked to Kurt Gödel incompleteness theorem.

The de facto hipster/web developer is a normal guy who went to a "boot camp" and learned JS and living happily ever after.

The problem with the two types of guys above is that they don't even know what a computer is. and given the fact above these are the guys whose jobs are to provide the correct instructions to the machine to perform a specific task. no wonder that our programs are as slow as ever despite the huge hardware improvements. Sometimes these programs even slower than their historical counterpart despite the fact that they are running on a super computers compared to the hardware in the 90s.

### Programming Style

Given the info above we can conclude easily that the native programming style of a computer is simply **Procedural Programming**. but there's a problem the prevalent programming style is the ideology of **OOP** and that doesn't follow our theory that Procedural Programming is the native programming style. In fact Procedural Programming was the dominant programming style but there was a problem that it had this problem is **Shared State** so to solve this problem another two styles were introduced **OOP** and **Functional Programming**.

#### The ideology of OOP

OOP was a response to the problem of the shared state of procedural programming. in fact it was quite an idiotic solution. it was no solution just a wrapping around the shared state. OOPs corner stone is **Encapsulation** and this is the OOP answer to shared state. the answer was simple you don't have shared state because you are going to encapsulate it in an object.

The idiocy of **Encapsulation** is quite apparent if you think with a critical mindset about it's ideas. many programmer doesn't think about it that's why I refer to it always as an ideology a cult or religion of some sort.

So let's start with some good old OOP. what is an object? an object is simply a bundle of data which represents that state of this object and a bundle of functions which represents the behaviours of this object. and then there's this childish notion that everything in the world is an object -popularized by java- ... I cannot comment about this childish fantasy but No, Carol many things around you are not objects for example `load_file` this is a simple function that loads a file in memory, `cross_product` this is a simple mathematical function. you don't need a `file_loader` in your life Carol.

Given the above info let's imagine a simple configuration:

```c++
class A
{
  int data;
  public:
  void update()
  {
    data++;
  }
};
```

This is object A. it's a simple object and only has an int that represents it's state and only one function that's called update and it's so simple it doesn't even take any input nor does it output anything at all. it just does one simple thing it increments the private data.

Now imagine in an application that you have another objects let's say objects B, C, and D. they all by some way or another call the function update of the object A ... either by having a reference to A or sending an event or by any other means. Can you tell me then what have the Encapsulation introduced/solved in this case. Simply nothing ... Zero. **Encapsulation doesn't work**.

#### Functional Programming

Functional Programming may sound reasonable. after all it's based on maths which should be reasonable. so it solves the problem of shared state simply by minimizing the shared state. which is a good solution actually but there's a problem with that solution if you go to the extreme. given the definition of the computer itself. the sole job of a computer is to load data into it's memory then edit this data by some logic thus transforming it into the output. so the idea that functions should not alter the state is simply bad and inefficient.

If you are going to copy everything around this will cost you a lot and if you are going to eliminate this from your language then you are rendering your language as inefficient and useless since it ignores the real nature and what the computer was designed to do in the first place.

#### Our Approach

Our approach is simple we minimize the state as much as possible. but if you are writing an application then you must have state left. these states will be encapsulated in a structure of some sort and this structure will represents that state of this area of the application at a time point.

- No classes ... only structs
- Minimize member functions unless the behaviour is highly bundled with the type itself
- No data hiding ... thus no private data everything is public and the data or functions that starts with an `_` underscore is used to indicate that you should not fiddle with these things but at the end of the day if you know what you are doing then feel free to use them
- Parameterize where possible
- Be explicit as much as possible. for example there's no `sort` function instead there's `merge_sort`,  `quick_sort`, `insertion_sort`, and `heap_sort`

## Memory

All memory interactions in CPPrelude is formalized inside what we call a slice, a slice consists of two simple things a pointer of type T and a size in bytes. each interaction with the memory either accepts a slice or outputs a slice.

C++ memory API is poor it simply consists of two functions that work in two modes singular mode and array mode. What if I wanted to resize my already allocated memory ? you would allocate another larger or smaller memory and copy/move the data into it.

C++ simply doesn't provide realloc functionality. C++ also doesn't provide a way to only allocate memory. new operator in C++ always invokes the constructor after allocating memory. which you may not want to do anyway so simply C++ bundles the idea of memory allocation and "object" initialization.

So I had to fallback to good old C memory API. except the C memory API has a major flaw. have you ever noticed that when you use `malloc` you specify the size and get a `void*` back. Ok fair enough, but when you `free` you simply provide the pointer back so how the system knows which amount of memory it should free you could have allocated 1 byte or 1 Gigabyte.

C memory API simply has an info table about the sizes of allocated memory and when you `free` it lookup the size of allocated memory this pointer points to and free this memory. but that's awful and is even more awful when you think about this pattern `do_some_work(void* ptr, std::size_t size)` each time you want to do any work on this memory you have to send the pointer and also don't forget about the size of memory this pointer points to so a slice was a good idea and avoids this draw back it bundles three things together a pointer that points to allocated memory, a type which identify the type of the data in this memory and also the size of this memory in bytes you could pass it around and use it like an array or downgrade it to a pointer do whatever you like but when you need to `free` it it's your responsibility to return it to its original type a slice.

So we went with the C memory API as a backend for now. and our slice interface lay above it.

### Automatic Memory Management

No automatic memory handling is implemented. I know that some people -of the two types above- can't live without a GC. I simply find it quite easy to manage my memory manually.

My experience with manual memory management is a happy one simply this scenario happens too much.
I code a big chunk of code that manually manages the memory then run it through Valgrind or VLD only to find "No memory leaks" and quite sometimes I find myself leave a memory intentionally to check if the memory leak checking program is even working at all. So manual memory management is simple done in the back of my mind all the time. I know what resources my program uses and when it needs it exactly.

Let's imagine the perfect memory management for your program. It knows when and the how much memory your program needs and performs memory management perfectly. Now after you have imagined this I have some questions for you.

- What's the probability that your GC be this perfect memory management solution for your program?
  - Simply it is approaches 0% ... GC is a generic program and any specialized version of it will beat it in terms of performance.
- What's the probability that your manual memory management system will be the perfect solution to your program?
  - It has a good probability if you the programmer know when, and how much memory your program need.

**P.S.**: I'm not bragging. many programmers know how to handle memory manually and all programming should know how to handle resources manually.

## Safety

The claim that you all your objects has to be in a well defined state at all times is false. so we allow our data and structure to exist in an unknown state given the fact that we will not check it at all. We will first transform it into a well defined state then we will check it ... so this freedom allows the code to be much simpler.

A programmer is a responsible human being. and my sole purpose is to provide him with the most efficient, most suitable, and sharpest tool possible. I will not provide dull tools in fear that a child may get hurt if you feel that you are an incapable programmer maybe learn more.

## Profit

The result of the combination of the above points is that our containers is faster by orders of magnitude than the bloated C++ STL Containers, more useful, and open for you to do whatever you want feel free to edit the structure internals or steal their data completely. This is your machine, your data, your program and your code. you are a responsible human-being not a child that must be taken care of out of the fear that he will hurt himself. here are some results

**Results**

| Container                                | STL                  | CPPrelude             |
| ---------------------------------------- | -------------------- | --------------------- |
| [debug] vector: pushing 10000 elements   | 5.17927 milliseconds | 0.541682 milliseconds |
| [release] vector: pushing 10000 elements | 51.6864 microseconds | 26.7719 microseconds  |
| [debug] queue: pushing 10000 elements    | 7.22153 milliseconds | 0.758483 milliseconds |
| [release] queue: pushing 10000 elements  | 155.358 microseconds | 14.0203 microseconds  |
| [debug] unordered_map: inserting, searching and removing 10000 elements | 166.495 milliseconds | 15.0626 milliseconds  |
| [release] unordered_map: inserting, searching and removing 10000 elements | 1255.53 microseconds | 328.278 microseconds  |