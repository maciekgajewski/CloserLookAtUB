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