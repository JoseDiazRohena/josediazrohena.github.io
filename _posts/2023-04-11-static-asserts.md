And so continues our crusade against warnings.

In this case, most of our warnings were implicit `float` to `double`
conversion. These were mainly resolved by amending our function interfaces to accept `float`. 

One exception is the `hyperTanFirstAntiDeriv`. This function has been
the source of overflows for a while now. Earlier I figured that the `cosh`
call was the culprit, as that function blows up for silly large numbers.
Because of this, I implemented a wrapper function that first clips input
that would cause an overflow.

One thing I neglected to consider is that the range to which input is clipped
is determined by the type in use. Originally, this was all implemented using
`double`. These changes, however, changed everything to `float`, as that is
the type we expect from calling code (maybe we can template that some time).
Doing this brought the overflows, NaN, and denormals back. 

So, I figured I could use different ranges for `float` and `double`. More about that
below. This work ended up solving the issue. `float` was now working without 
overflow...most of the time. I got another  CI failure in Windows, and figured that it 
would be best to err on the side of caution and just cast  to `double` when doing 
the antiderivative and casting back to `float` on the way out.

Nevermind the fancy `constexpr` magic  I worked to "cleanly" set those 
`clampedCosine` bounds. I learned a lot about templates, specifically how
branching works for `if constexpr` expression.

Here's what I wanted to do: create a function that returned a pair of 
float types, depending on the function's type:

```
template <typename T>
constexpr std::pair<T, T> coshRange()
{
    if constexpr (std::is_same<T, double>::value)
    {
        return { range };
    }
    else if constexpr (std::is_same<T, float>::value)
    {
        return { a different range };
    }
    else
    {
        static_assert (false, "ClampedCosh only supports float types");
    }
}

```

I was extremely confused when I saw the `static_assert` failing, even when instantiating a 
function with an accepted type.

Some searching lead me to this answer on [Stack Overflow](https://stackoverflow.com/questions/68526152/c-static-assert-fails-on-both-branches-of-an-if-constexpr-statement), which I understand as meaning: a `static_assert` is always compiled. Therefore,
writing something like `static_assert(false,"message")` will always fail.

The solution is to, with the help of `type_traits`, create a helper type that always evaluates to false and, most importantly, only is compiled when instantiated:

```
template <typename T>
struct dependent_false : std::false_type
{
};
```

Now we can write that else branch as:

```
static_assert (dependent_false<T>, "ClampedCosh only supports float types");
```

And the static_assert will correctly (successfully?) fail if the function was instantiated with an 
incompatible type.

After all of this, I ended up not using the specialization that all this fuss was for—as I mentioned above,
I still got some overflows and ended up doing that part of the calculation with `double`. But 
it was a fun bit of learning that I've chosen to keep in.
