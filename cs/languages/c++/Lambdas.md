**Лямбда-функция** (или лямбда-выражение) — это анонимная функция, которая может быть определена и использована на месте без необходимости объявления отдельной именованной функции.

``` C++  
[ captures ] ( params ) specifiers exception attr -> ret { body }
```

- `[]` - Список захвата, определяет внешние объекты, которые доступны внутри лямбда-функции (capture clause)
- `()` - список параметров (опционально)
- спецификаторы (`mutable`,`noexcept`,`attributes`, `constexpr`...) (опционально)
- `ret` - тип возвращаемого значения (опционально)
- `{ body }` - Тело лямбда-функции.


Примеры:

``` cpp
[x](int x, int b) mutable { ++x; return a < b};
[](float param) noexcept { return param*param };
[x](int a, int b) mutable noexcept { x++; return a < b; };
```

---
## Closure type

Каждое лямбда-выражение в C++ создаёт уникальный тип, который называется **типом замыкания (closure type)**

Объект этого типа представляет из себя экземпляр функтора, который автоматически генерирует компилятор и который содержит код лямбда-функции и захваченные объекты в качестве своих полей.


``` cpp
auto lambda = [x]() { std::cout << x << std::endl; };

// this lambda can be represented as:
class _lambda_12_17 {
private:
    int x;

public:
    _lambda_12_17(int x) : x(x) {}

    inline void operator()() const {
        std::cout << x << std::endl;
    }
};
```

При этом само лямбда-выражение является `prvalue` с точки зрения категории выражений C++.

Согласно стандарту, тип замыкания не имеет конструктора по умолчанию, а также не имеет оператора присваивания. Однако копирующий конструктор для лямбда-выражений определён.

``` cpp
auto lambda_1 = [](int x) { return x * x; };
auto lambda_2 = lambda_1; // fine
```

---
## Captures

Захват объектов - это механизм, который позволяет лямбда-выражению использовать объекты из окружающей области видимости.

Захваченные объекты перечисляются в фигурных скобках лямбда-выражения `[ ]`

Различные варианты захвата объектов:

``` cpp
int x = 2, y = 3;

// No objects capture
const auto l_1 = []() { return 1; };

// capture all objects by value (copy of 'x' is used inside)
const auto l_2 = [=]() { return x; };

// capture all objects by reference
const auto l_3 = [&]() { return y; };

// Explicitly capture object 'x' by value
const auto l_4 = [x]() { return x; };

// capture all objects by value, except 'x' by reference
const auto l_5 = [=, &x]() { return x + y; };

// capture a template argument pack, all by reference
template<typename... Args>
void foo(Args... args) {
	auto lambda = [&args...]() {
		int index = 0;
		((std::cout << "Argument " << index++ << ": " << args << std::endl), ...);
	};
	lambda();
}
```

### Init Captures

**Init captures** (инициализаторы захвата) — это расширение стандарта C++14, которое позволяет инициализировать захваченные объекты прямо в списке захвата лямбда-выражения.

``` cpp
std::string str = "Hello";
auto lambda = [s = str + " World"]() { return s; };
std::cout << lambda() << std::endl;  // Output is "Hello World"

std::vector<int> vec = {1, 2, 3};
auto lambda = [size = vec.size()]() { return size; };
std::cout << lambda() << std::endl;  // Output is 3
```


Также с помощью инициализаторов захвата была добавлена возможность перемещать объекты в лямбда-выражение при захвате:

``` cpp
auto lambda = [p = std::move(ptr)]() {
    if (p) {
        std::cout << "Value: " << *p << std::endl;
    } else {
        std::cout << "Pointer is null" << std::endl;
    }
};
```

---
## Capturing a Class Member and the `this` pointer

Представим ситуацию: в методе класса необходимо захватить поле этого класса в лямбда-выражение.

``` cpp
class Example {
private:
	std::string s;
public:
	void foo() {
		const auto lam = [s]() { std::cout << s; };
		lam();
	}
};
```

Однако данный код вызовет ошибку:

``` clang
In member function 'void Example::foo()':
error: capture of non-variable 'Example::s'
error: 'this' was not captured for this lambda function
```

Для решения этой проблемы необходимо захватить указатель `this`.
Это даст доступ лямбда-выражению ко всем данным этого класса.

``` cpp
class Example {
private:
	std::string s;
public:
	void foo() {
		const auto lam = [this]() { std::cout << s; };
		lam();
	}
};
```

---

## Conversion to a function pointer

В некоторых случах бывает нужна возможность использовать лямбда-выражения в местах, где ожидаются указатели на функции. В этом случае лямбда-выражения могут быть преобразованы в указатели на функции.

Данное преобразование возможно только в том случае, если лямбда-выражение не захватывает никаких объектов из окружающей среды.

``` cpp
int (*func_ptr)(int, int) = [](int a, int b) { return a + b; };


int result = func_ptr(3, 4);  // result = 7
```

Этот механизм предоставляет совместимость классический сишных указателей и лямбда-выражений, соответсвующих по сигнатуре.

---
## IIFE

Immediately invoked functional expression - это способ вызывать лямбда-выражение в месте определения.

``` cpp
[&]() noexcept { ++x; ++y; }(); // <-- call()

int result = [](int a, int b) -> int { return a + b; }(2, 3);
```

В случае, когда функция достаточно объёмная и не слишком читабельная, можно использовать замену IIFE - функцию `std::invoke()` из стандартной библиотеки.

``` cpp
int result = std::invoke([](int a, int b) { return a + b; }, 2, 3);
```

---
## Generic Lambdas

В C++14 появилась возможность создавать лямбда-выражения, параметры которых могут быть **шаблонными**. Это достигается с помощью ключевого слова `auto` в списке параметров:

``` cpp
auto genericLambda = [](auto x) { return x * x; };
```

это эквивалентно использованию шаблонного аргумента в соответсвующем данному лямбда-выражению функциональному объекту:

``` cpp
class __lambda_functor {
public:
    template<typename T>
    auto operator()(T x) const {
        return x * x;
    }
};
```

В generic lambdas возвращаемый тип может быть выведен автоматически, если тело лямбды состоит из одного оператора `return`. В противном случае, нужно явно указать тип возвращаемого значения:

``` cpp
auto complexLambda = [](auto x) -> decltype(x * x) {
    if (x > 0) {
        return x * x;
    } else {
        return -x * x;
    }
};
```

### Variadic generic arguments

Вариативные шаблонные аргументы позволяют лямбда-выражениям принимать произвольное количество аргументов разных типов. 

Синтаксически это выглядит как использование `auto...` в списке параметров лямбды:

``` cpp
auto applyFunction = [](auto func, auto... args) {
    return func(args...);
};

auto square = [](auto x) { return x * x; };

int result = applyFunction(square, 2, 3, 4); 
// result = 16 (square of the last element)
```


Также вариативные шаблонные аргументы в лямбда-выражениях можно совмещать с использованием **fold expressions**:

``` cpp
auto sumArgs = [](auto... args) -> decltype((args + ...)) {
    return (args + ...);
};
```

---
## std::function<>

`std::function` — это шаблонный класс из стандартной библиотеки, который может хранить и вызывать функциональные объекты любого типа, соответствующего заданной сигнатуре:

``` c++
#include <functional>

std::function<int(int)> func;
```

Здесь `std::function<int(int)>` — это обертка для функционального объекта, который принимает один аргумент типа `int` и возвращает значение типа `int`.


Лямбда-выражения могут быть использованы для инициализации объектов `std::function`. 
Это позволяет хранить лямбда-выражения в переменных типа, передавать их в функции и возвращать из функций:

``` c++
#include <functional>

std::function<int(int)> func = [](int x) { return x * x; };

int result = func(5);  // result = 25

-----------------------------------------------------------

#include <functional>
#include <iostream>

std::function<int(int)> createMultiplier(int factor) {
    return [factor](int x) { return x * factor; };
}

int main() {
    auto multiplyByTwo = createMultiplier(2);
    std::cout << multiplyByTwo(5) << std::endl;  // Output: 10
}
```


Также использование `std::function` позволяет определить рекурсивные лямбда-выражения:

``` c++
int main() {
	const std::function<int(int)> factorial = [&factorial](int n) {
		return n > 1 ? n * factorial(n - 1) : 1;
	};
	return factorial(5);
}
```


Хотя `std::function` предоставляет удобный интерфейс для работы с функциональными объектами, он имеет некоторый оверхед по сравнению с использованием лямбда-выражений напрямую. Это связано с тем, что `std::function` должен быть способным хранить объекты разных типов, что требует динамического выделения памяти и дополнительных проверок во время выполнения.

В качестве альтернативы можно использовать шаблонные функции или классы, но это увеличивает загромождённость кода:

``` c++
#include <functional>
#include <iostream>

void applyAndPrint(std::function<int(int)> func, int value) {
    std::cout << func(value) << std::endl;
}

int main() {
    applyAndPrint([](int x) { return x * x; }, 5); 
}

---------------------------------------------------------------
// alternative way
template<typename Func>
void applyAndPrint(Func func, int value) {
    std::cout << func(value) << std::endl;
}

int main() {
    applyAndPrint([](int x) { return x * x; }, 5);
}
```

---
