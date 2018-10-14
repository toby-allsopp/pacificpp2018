---
theme: simple
highlightTheme: idea
revealOptions:
    transition: none
    transitionSpeed: fast
    controls: false
    slideNumber: 'c'
    width: '100%'
    margin: 0
    center: false
---
<!-- .slide: class="center" -->
# Surfacing Composition

### Toby Allsopp
WhereScape Software Limited

`toby@mi6.gen.nz`<br/>
Twitter: `@toby_allsopp`

Notes: Please feel free to interrupt me to ask any questions you have, or wait
until the end and we'll see how much time is left.

---
<!-- .slide: class="center" -->

# Composition

<p class="fragment">*closure*: for any integers $x$ and $y$, $x + y$ is an integer</p>

Notes: Composition is a fairly overloaded term in programming. I'm going to talk
about it in a sense that's both very general and quite specific. I'm going to
talk about composition as a kind of algebraic structure, meaning we have a set
of something and one or more operations on elements of that set that result in
other elements of the set.

A, hopefully familiar, example is the set of integers and addition - addition is
an operation that takes two integers and results in a third. No matter which two
integers you feed into the addition operation, you always get an integer out.
This property is known as **closure**.

In the kind of composition I'm going to talk about, the elements of the set are
parts of your program, for example expressions, statements or functions, and the
operations are things that we, as programmers, do to them.

---
<!-- .slide: class="center" -->

# Expressions

Notes: Expressions in most programming languages, including C++, exhibit rich
composition. We can start with a relatively small set of primitive expressions
and compose them together to make an infinite set of possible expressions.


---

## Primitive Expressions

-   **literal:** `3` or `"Hello"`
-   **id-expression:** `x`

## Operations on Expressions

-   **parenthesise:** `(x)`
-   **unary operator:** `-x`
-   **binary operator:** `x + 4`
-   **ternary operator:** `x ? 7 : y`
-   **function call:** `f(x, y)`

---

## Restrictions

-   `"Hello"s + 7` (no matching overload)
-   `&7` (can't take the address of a prvalue)
-   `7(1, 2)` (not a function)

Not every composition yields an expression, but none yield anything *other* than
an expression.

Notes: Now, not all combinations of expressions are _legal_ in C++ - we have to
worry about types and value categories and what-not.

---
<!-- .slide: class="center" -->

# Statements

Notes: Statements in C++ have much more limited composition possibilities than
expressions.


---

## Primitive Statements

-   expression statement
    - `x + 4;`
-   jump statement
    - `goto fail;`
    - `return 7;`
-   declaration statement
    - `auto x = 5;`

---

## Composed Statements

-   compound statement
  - `{` *statement<sub>1</sub>* &#x2026; *statement<sub>n</sub>* `}`
-   selection statement
  - `if` `(` *condition* `)` *statement<sub>1</sub>* `else` *statement<sub>2</sub>*
  - `switch` `(` *condition* `)` *statement*
-   iteration statement
  - `do` *statement* `while` `(` *condition* `)`;
  - `while` `(` *condition* `)` *statement*
  - `for` `(` *for-stuff* `)` *statement*

---
<!-- .slide: class="center" -->

# Types

Note:

---

## Built-in Types

- `char`
- `int`
- `double`

## Compound Types

- `struct` - product
- `union` - sum
- `enum class` - "copy"

Note:
That's about it for ways to define new types in C++.

---

Notes: This is, so far, composition built into the grammar of the language.
There's no way to change or extend this - you can't introduce a new kind of
expression or statement.

Things get more interesting when we consider functions.


---
<!-- .slide: class="center" -->

# Functions

Notes:
Can we combine functions together to make other functions? Of course we can!


---

## Manual Function Composition

```c++
struct person {
  year_month_day dob;
  std::string name;
};

int dob_to_age(date dob) {
  return floor<years>(system_clock::now() - dob).count();
}

int person_age(person p) {
  return dob_to_age(p.dob);
}
```

---

## Surfaced Function Composition

$\mathtt{compose}(f, g) = \lambda (x) \rightarrow f(g(x))$

<pre class="fragment"><code class="c++" data-trim data-noescape>
template &lt;typename F, typename G>
auto compose(F f, G g) {
  return [=](auto x) {
    return <span class="fragment highlight-current-green">std::invoke(</span>f, std::invoke(g, x));
  };
};
</code></pre>

<table width="90%">
<tr>
<td>
<pre class="fragment"><code class="c++" data-trim data-noescape>
int person_age(person p) {
  return dob_to_age(p.dob);
}
</code></pre>
</td><td>
<pre class="fragment"><code class="c++" data-trim data-noescape>
int person_age(person p) {
  return compose(dob_to_age, &person::dob)(p);
}
</code></pre>
</td>
</tr>
</table>

<pre class="fragment"><code class="c++" data-trim data-noescape>
<span class="fragment strike">inline constexpr auto person_age = compose(dob_to_age, &person::dob);</span>
</code></pre>

But is that actually useful?
<!-- .element: class="fragment" -->

---

## Manual Function Composition

```c++
struct person {
  year_month_day dob;
  std::string name;
};
int person_age(person p);
```

```c++
bool any_wise_ones(std::vector<person> people) {
  return std::any_of(begin(people), end(people),
                     [](person p) { return person_age(p) > 40; });
}
```
---

## Surfaced Function Composition

```c++
auto greater_than(int x) {
  return [=](auto y) { return y > x; };
}

bool any_wise_ones(std::vector<person> people) {
  return std::any_of(begin(people), end(people),
                     compose(greater_than(40), person_age));
}
```

---

## Surfaced Function Composition

$\mathtt{partial}(f, x) = \lambda (x_1, \dots) \rightarrow f(x, x_1, \dots)$

```c++
auto greater_than(int x) {
  return std::bind(std::greater(), _1, x);
}

bool any_wise_ones(std::vector<person> people) {
  return std::any_of(begin(people), end(people),
                     compose(greater_than(40), person_age));
}
```

<p class="fragment">OK, but is it *really* useful?</p>

---

## More Complicated Function Composition

```c++
void sort_by_age(std::vector<person>& people) {
  std::sort(begin(people), end(people),
            [](person p1, person p2) {
              return p1.age < p2.age;
            });
}
```

```c++
void sort_by_name(std::vector<person>& people) {
  std::sort(begin(people), end(people),
            [](person p1, person p2) {
              return p1.name < p2.name;
            });
}
```
<!-- .element: class="fragment" -->

---

## Surfaced Function Composition

$\mathtt{on}(f, g) = \lambda (x_1, x_2, \dots) \rightarrow f(g(x_1), g(x_2), \dots)$

```c++
template <typename F, typename G>
auto on(F f, G g) {
  return [=](auto... x) {
    return std::invoke(f, std::invoke(g, x)...);
  };
}
```
<!-- .element: class="fragment" -->

```c++
void sort_by_age(std::vector<person>& people) {
  std::sort(begin(people), end(people),
            on(std::less(), &person::age));
}

void sort_by_name(std::vector<person>& people) {
  std::sort(begin(people), end(people),
            on(std::less(), &person::name));
}
```
<!-- .element: class="fragment" -->

Notes: This is a generalisation of `compose`

---
$\mathtt{compose}(f, g) = \lambda (x) \rightarrow f(g(x))$


$$\mathtt{on}(f, g) = \lambda (x_1, x_2, \dots) \rightarrow f(g(x_1), g(x_2), \dots)$$

\begin{align}
  \mathtt{on}(f) &= f \\\\
  \mathtt{on}(f, g) &= \lambda (x_1, x_2, \dots) \rightarrow f(g(x_1), g(x_2), \dots) \\\\
  \mathtt{on}(f_1, f_2, \dots, f_n) &= \lambda (x_1, x_2, \dots) \rightarrow f(f_n(x_1), g(x_2), \dots)
\end{align}

---

## More Difficult Composition

<pre style="float:left; width: fit-content"><code class="c++" data-noescape data-trim>
struct person {
  std::optional&lt;date> dob;
  std::string name;
};
<span class="fragment">
int dob_to_age(date dob) {
  return today() - dob;
}</span>
<span class="fragment">
std::optional&lt;int>
person_age(person p) {
<mark>  if (!p.dob) return std::nullopt;
  return dob_to_age(*p.dob);</mark>
}
</span>
</code></pre>

<!-- style="float: left; width: fit-content;" -->

<pre style="float:right; width: fit-content"><code class="c++" data-noescape data-trim>
<span class="fragment">
bool is_age_wise(int age) { return age > 40; }</span>
<span class="fragment">
std::optional&lt;bool> is_person_wise(person p) {
  const auto age = person_age(p);
<mark>  if (!age) return std::nullopt;
  return is_age_wise(*age);</mark>
}</span>
<span class="fragment">
bool any_known_wise_ones(std::vector&lt;person> people) {
  return std::any_of(begin(people), end(people),
                     [](person p) {
                       return is_person_wise().value_or(false);
                     });
}</span>
</code></pre>

---

## Composing `std::optional`

```c++
const auto compose_optional = [](auto f, auto g) {
  return [=](auto x) -> std::optional<decltype(std::invoke(f, *std::invoke(g, x)))> {
    const auto o = std::invoke(g, x);
    if (!o) return std::nullopt;
    return std::invoke(f, *o);
  };
};
```

```c++
const auto person_age = compose_optional(dob_to_age, &person::dob);

const auto is_person_wise = compose_optional(is_age_wise, person_age);

const auto value_or = [](auto x) {
  return [=](auto o) { return o.value_or(x); };
};

bool any_known_wise_ones(std::vector&lt;person> people) {
  return std::any_of(begin(people), end(people),
                     compose(value_or(false), is_person_wise));
}
```

---
<!-- .slide: class="center" -->

# Thank you!

Questions?

`toby@mi6.gen.nz`<br/>
Twitter: `@toby_allsopp`
