class: center, middle

# Closer Look 
## at Undefined Behaviour and Complier Optimizations

Maciej Gajewski

---

# About me

* Maciek Gajewski [maciej.gajewski0@gmail.com](mailto:maciej.gajewski0@gmail.com)
* I work at Optiver, Amsterdam
* Role: C++ Developer and teacher

<img src="pics/Maciek.jpg" height="200"/>
<img src="pics/optiver_logo_black.png" height="100"/>

---

# Undefined Behavior
### The popular definition

> “When the compiler encounters [a given undefined construct] it is legal for it to make demons fly out of your nose”

comp.std.c, 1992

???

Completely useless from educational point of view.

---
# Undefined Behavior
### More useful definition

* Machine-dependent behavior that would be too costly to define
* Something, that compiler can assume you would never do


---
# Undefined Behavior

> The essence of undefined behavior is the freedom to avoid a forced coupling between error checks and unsafe operations. 

> In summary, undefined behavior in programmer-visible abstractions represents an aggressive and dangerous tradeoff: it sacrifices program correctness in favor of performance and compiler simplicity. 

[https://blog.regehr.org/archives/1467]

???
Too long to read during the presentation


---
# Let's have a closer look

* Compiler Explorer: https://godbolt.org
* By Matt Gotbolt
* CppCon 2017 _“What Has My Compiler Done for Me Lately?”_


TODO: screenshot?

???
Survey the auodience: who knows compiler expored.
Great tool for teachers and tweakers
Basic assembly required


---
# Part 1
## Null-pointer dereference

> dereferencing a null pointer is undefined behavior

---

### If dereferencing nullptr was defined...

This C code...
```cpp
size_t fread(void* buf, size_t size, FILE* s)
{
	if(s->bsize = 0) {
		s->bsize = (*s->vtbl->read)(s->buf, s->bufsize, s->fd);
	}
	return copy_from_buffer(buf, size, s);
}
```

would really be this:
```C
size_t fread(void* buf, size_t size, FILE* s)
{
	if (!s) __raise_error();
	if(s->bsize = 0) {
		if (!s->vtbl || ! s->vtbl->read) __raise_error();
		s->bsize = (*s->vtbl->read)(s->buf, s->bufsize, s->fd);
	}
	return copy_from_buffer(buf, size, s);
}
```

???
Compiler would be forced to generate code checking for null pointer every time a pointer is accessed.

---
# Some code
.pull-left[

### Left column

```C++
int fun(int i)
{
	int array[4];
	array[i] = 333;
	return array[i];
}
```]

.pull-right[

### Right column

```C++
int fun(int)
{
	return 333;
}
```]