*C++* is an *OOP* language thus it implements *Polymorphism* using *Inheritance*. But the problem in doing so is that extending other types becomes intrusive to the type definition itself, and what about extending other already compiled library types?, or what about re-routing the behaviour to a non-type entity?. Well, all of these options become impossible unless you are willing to do many adapters. In this article we explore other solutions to this problem and introduce what *CPPrelude* does to achieve it.

To View in PDF format [click here.](https://github.com/MoustaphaSaad/MoustaphaSaad.github.io/raw/master/_posts/cpprelude-polymorphism.pdf)

## Type Systems

### Introduction

To a machine there are no types. Data is data. Types are language constructs to assist the programmer. But sooner or later the need will arise to write a generic function that could act upon many types or types that has a specific property or layout.

### Type Systems Options

Let us first talk about type systems to better understand the problem. Type systems can be vary along two axis orthogonally. The first axis: (dynamic, static), and the second axis: (strong, weak)

|             |  Strong  | Weak  |
| :---------: | :------: | :---: |
| **Static**  |  *C++*   |  *C*  |
| **Dynamic** | *Python* | *PHP* |

Since *C++* is a strongly statically typed language it had to solve the problem with polymorphism, and it did that using two methods: (*Templates*, *Inheritance*). However templates were not an intended solution to the problem but then people shoehorned them as solution.

```C++
static usize
foo(?? writable)
{
	return writable->write(...);
}
```

- The strongly statically typed language problem is the input type of function foo.

What should you substitute the **??** with?. This is the problem. If you are using a dynamically typed language you wouldn’t need to specify the type in the first place.

```python
def foo(writable):
	return writable.write(...)
```

- A Dynamically typed language like python doesn’t specify the input type of function foo.

You should realize by now that nobody actually needs inheritance. But everybody needs polymorphism and code reuse.

## Solutions

### Inheritance Solution

```C++
struct IWritable
{
	virtual usize write(const slice<byte>&) = 0;
};

struct buffer: IWritable
{
  	usize write(const slice<byte>&) override;
};

static usize
foo(IWritable *writable)
{
  	return writable->write(...);
}
```

- Inheritance way of achieving polymorphic use of function foo.

The problem with inheritance solution is that it’s intrusive to the type and when you deal with an external/library type you don’t have the luxury of editing it.

### Template Solution

```C++
struct buffer
{
  	usize write(const slice<byte>&);
};

template<typename T>
static usize
foo(T *writable)
{
  	return writable->write(...);
}
```

-  Template/Generic way of achieving polymorphic use of function foo

The problem with template solution is that it can and will accept any type you pass to the function thus acting as if it removed a big chunk of any decent type system features and in addition to that you get weird error messages.

### Go and Rust Solution

```go
type IWritable interface {
  	Write(slice) int
}

type Buffer struct {
}

func (buf Buffer) Write() int {
  	...
}

func foo(IWritable writable) int
{
  	return writable.Write(...)
}
```

-  *Go* way of achieving polymorphic use of function foo

```Rust
struct Buffer;

trait IWritable {
  	fn write(&self, ...) -> usize;
}

impl IWritable for Buffer {
  	fn write(&self, ...) -> usize {
      	...
  	}
}

fn foo(writer: &mut IWritable) -> usize {
  	writer.write(...)
}
```

-  *Rust* way of achieving polymorphic use of function foo

Essentially Go and Rust solutions are the same. I actually like this solution. It separates your interface from the concrete types that implement this interface and since member functions could be added outside the type itself you can easily extend any type whether it’s a library type or internal type.

### CPPrelude Solution

```C++
struct writable_trait
{
  	using write_func = usize(*)(void*, const slice<byte>&);
  	void *_self = nullptr;
  	write_func _write = nullptr;
  
	usize
    write(const slice<byte>&)
    {
      	return _write(_self, ...);
    }
};

struct Buffer
{
  	writable_trait _write_trait;
  
	static usize
    _buffer_write(void* _self, const slice<byte>&)
    {
      	...;
    }
  
	Buffer()
    {
		_write_trait._self = this;
      	_write_trait._write = &Buffer::_buffer_write;
	}
  
	inline operator writable_trait*()
    {
      	return &_write_trait;
    }
  	
  	usize write(const slice<byte>&)
    {
      	return _buffer_write(this, ...);
    }
};

usize
foo(writable_trait* writable)
{
  	return writable->write(...);
}

```

-  *CPPrelude* way of achieving polymorphic use of function foo

This way the anyone can extend the the trait for internal types and library types also you can provide the trait without associating it to any type at all as illustrated below.

```C++
//dummy function that writes nothing
usize
my_null_writer(void*, const slice<byte>&)
{
  	return 0;
}

void
without_type()
{
  	writable_trait dummy;
  	dummy._write = my_null_writer;
  	foo(&dummy);
}

void
with_type()
{
  	Buffer buf;
  	foo(buf);
}
```

-  *CPPrelude* using function foo without any associated types