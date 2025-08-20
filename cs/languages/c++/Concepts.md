// TODO

...

``` C++
#include <iostream>
#include <concepts>
#include <type_traits>


template <typename F, typename T>
concept Additive = requires(F f, T x, T y) {
    { f(x + y) } -> std::same_as<T>;
    { f(x) + f(y) } -> std::same_as<T>;
};


template <typename F, typename T>
concept Homogeneous = requires(F f, T x, typename T::value_type a) {
    { f(a * x) } -> std::same_as<T>;
    { a * f(x) } -> std::same_as<T>;
};


template <typename F, typename T>
concept LinearOperator = Additive<F, T> && Homogeneous<F, T>;
```