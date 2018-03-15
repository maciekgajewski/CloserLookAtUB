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

<!-- ====== Null pointer ======== -->

---
class: center, middle
# Chapter 1
## Null-pointer dereference

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
--
would really be this:
```c
size_t fread(void* buf, size_t size, FILE* s)
{
	`if (!s) __raise_error();`
	if(s->bsize = 0) {
		`if (!s->vtbl || ! s->vtbl->read) __raise_error();`
		s->bsize = (*s->vtbl->read)(s->buf, s->bufsize, s->fd);
	}
	return copy_from_buffer(buf, size, s);
}
```

???
Compiler would be forced to generate code checking for null pointer every time a pointer is accessed.
And if this is not C++ enough....

---

### If dereferencing nullptr was defined...

This C++ code...
```cpp
size_t File::read(void* buf, size_t size)
{
	if(bsize = 0) {
		bsize = do_read(buf, bufsize, fd);
	}
	return copy_from_buffer(buf, size);
}
```
would really be this:
```c
size_t File::read(File* this, void* buf, size_t size)
{
	`if (!this) __raise_error();`
	if(this->bsize = 0) {
		`if (!this->vtbl || ! this->vtbl->do_read) __raise_error();`
		this->bsize = (*this->vtbl->do_read)(this, this->buf, this->bufsize, this->fd);
	}
	return copy_from_buffer(this, buf, size);
}
```

???
But compilers are not doing that!

---
### Dereferencing nullptr

.pull-left[
This code
```cpp
// Safe to call with nullptr
void fun(Widget* w) {
	if (w)
		do_smth(w->data);
	else
		report_error();
}

// 'w' can't be null!
void fun2(Widget* w) {
	w->data = get_data();
	fun(w);
}
```
]
--
.pull-right[
	after inlining
```cpp
// Safe to call with nullptr
void fun(Widget* w) {
	if (w)
		do_smth(w->data);
	else
		report_error();
}

// 'w' can't be null!
void fun2(Widget* w) {
	w->data = get_data();
	if (w)
		do_smth(w->data);
	else
		report_error();
}
```
]

???

first step - inlining
---
### Dereferencing nullptr

.pull-left[
This code
```cpp
// Safe to call with nullptr
void fun(Widget* w) {
	if (w)
		do_smth(w->data);
	else
		report_error();
}

// 'w' can't be null!
void fun2(Widget* w) {
	w->data = get_data();
	fun(w);
}
```
]
.pull-right[
	after inlining
```cpp
// Safe to call with nullptr
void fun(Widget* w) {
	if (w)
		do_smth(w->data);
	else
		report_error();
}

// 'w' can't be null!
void fun2(Widget* w) {
	w->data = get_data();
	`if (w)`
		do_smth(w->data);
	`else`
		`report_error();`
}
```
]

???

next step - removing null branch
---
### Dereferencing nullptr

.pull-left[
This code
```cpp
// Safe to call with nullptr
void fun(Widget* w) {
	if (w)
		do_smth(w->data);
	else
		report_error();
}

// 'w' can't be null!
void fun2(Widget* w) {
	w->data = get_data();
	fun(w);
}
```
]
.pull-right[
optimized
```cpp
// Safe to call with nullptr
void fun(Widget* w) {
	if (w)
		do_smth(w->data);
	else
		report_error();
}

// 'w' can't be null!
void fun2(Widget* w) {
	w->data = get_data();
	do_smth(w->data);
}
```
]

???

final stage

<!-- TODO: add the kernel bug -->


<!-- ====== Array ======== -->



---
class: center, middle
# Part 1
## Accessing array out of bounds

---
# Very simple example
.pull-left[
This code
```cpp
void fun(int idx, int val) {
	int array[3];
	array[idx] = val;
	return array[idx];
}
```
]
--
.pull-right[
is compiled to
```cpp
void fun(int, int val) {
	return val;
}
```
]

???
Seems like a contrived example, but the code may be a result of 
- inlining, 
- dead-branch remove, 
- compile-time evauation of constant expressions

---
# A subtle bug
.pull-left[
This code
```cpp
template<typename T, size_t N>
bool is_in_arr(array<T, N> arr, T v)
{
	for(int i = 0; i < N; i++)
		if (arr[i] == v)
			return true;

	return false;
}
```
]
---
# A subtle bug
.pull-left[
This code
```cpp
template<typename T, size_t N>
bool is_in_arr(array<T, N> arr, T v)
{
	for(int i = 0; i < N; i++)
		if (arr[i] == v)
			return true;

	return false;
}

bool is_right_angle(int deg)
{
	array<int, 4> arr 
		= {0, 90, 180, 270};
	return is_in_arr(arr, deg);
}```
]
--
.pull-right[
is compiled to
```asm
s_right_angle(int):
	test edi, edi
	je .L5
	cmp edi, 90
	je .L5
	cmp edi, 180
	je .L5
	cmp edi, 270
	sete al
	ret
.L5:
	mov eax, 1
	ret
```
]
---
# A subtle bug (2)
.pull-left[
This code
```cpp
template<typename T, size_t N>
bool is_in_arr(array<T, N> arr, T v)
{
	for(int i = 0; i `<=` N; i++)
		if (arr[i] == v)
			return true;

	return false;
}

bool is_right_angle(int deg)
{
	array<int, 4> arr 
		= {0, 90, 180, 270};
	return is_in_arr(arr, deg);
}```
]
--
.pull-right[
is compiled to
```asm
s_right_angle(int):
	mov eax, 1
	ret
```
]

