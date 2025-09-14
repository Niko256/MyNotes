
[Виртуальная таблица](https://llvm.org/devmtg/2021-11/slides/2021-RelativeVTablesinC.pdf) - это структура данных, расположенная в сегменте `data`, которая обеспечивает динамический полиморфизм (`dynamic dispatch`).

- Когда в классе объявляется хотя бы одна виртуальная функция, компилятор создает `vtable` в сегменте `data`.
	- для всех экземпляров класса создаётся единая виртуальная таблица.

	- производный класс и его базовый класс будут использовать общую `vtable` в случае, когда производный класс не переопределяет ни одну из виртуальных функций базового класса и сам не объявляет новых.

- Каждый объект такого класса содержит скрытый указатель на `vtable`, называемый `vptr` (`vtable adress point`).

---
### [Vtable Layout](https://refspecs.linuxfoundation.org/cxxabi-1.86#vtable) 

рассмотрим примитивный пример:

``` cpp
class A {
public:
	virtual void foo();
	virtual void bar();
};

class B : public A {
public:
	void bar() override;
};

void func(A* a) {
	a->bar();
}
```

как будет выглядеть `vtable` для класса `A`? (Itanium C++ ABI)



<img src=https://github.com/user-attachments/assets/b87366b5-1907-41e3-b8bd-0c9fcd4e6052 alt= vt width=400>

$$\space$$

| Component                        | Type                                          | Value                          |
| -------------------------------- | --------------------------------------------- | ------------------------------ |
| Offset to top                    | ptrdiff_t                                     | 0                              |
| Run-Time Type Information (RTTI) | pointer to struct `typeinfo`                  | some ptr to `data` segment<br> |
| virtual function `foo()`         | pointer to function `foo()` in `text` segment | `A::foo()`                     |
| virtual method `bar()`           | pointer to function `bar` in `text` segment   | `A::bar()`                     |


- `Offset to top` - это сдвиг, равный "расстоянию" в байтах до самого верхнего базового класса в иерархии наследования. может быть как положительным, так и отрицательным по значению.

данное представление относится только к самому примитивному случаю и немного отличается в случае виртуального и множественного наследования

про эти специфические случаи можно почитать здесь:

- https://shaharmike.com/cpp/vtable-part2/
- https://shaharmike.com/cpp/vtable-part3/
- https://shaharmike.com/cpp/vtable-part4/

---

## [RTTI](https://refspecs.linuxfoundation.org/cxxabi-1.86#rtti)

Run-Time Type Information - это механизм, который позволяет динамически определить тип полиморфного объекта на этапе `Run-Time`.

В C++ данный механизм реализуется с помощью отдельной [структуры](https://en.cppreference.com/w/cpp/types/type_info) `typeinfo`, хранящейся в сегменте `.data` и содержащей информацию о полиморфном типе, а также с помощью операторов `typeid` и `dynamic_cast<>`.

---
### `typeid` && `dynamic_cast`

Структура `typeinfo` используется такими операторами как `typeid` и `dynamic_cast<>`:

- **Оператор `typeid`**
	- Возвращает объект типа `std::type_info`, который содержит информацию о типе выражения.
		- Это позволяет сравнивать типы и получать строковое представление типа.

- **Оператор `dynamic_cast<>`**
	- Используется для безопасного приведения указателей и ссылок на полиморфные типы.
		- Если приведение невозможно, `dynamic_cast` возвращает `nullptr` для указателей или выбрасывает исключение `std::bad_cast` для ссылок.


``` c++
#include <iostream>
#include <typeinfo>

class Base {
public:
    virtual ~Base() {}
};

class Derived : public Base {
};

int main() {
    Base* b = new Derived;

    if (typeid(*b) == typeid(Derived)) {
        std::cout << "b is actually a Derived object!" << std::endl;
    }

    Derived* d = dynamic_cast<Derived*>(b);
    if (d) {
        std::cout << "Dynamic cast successful!" << std::endl;
    }

    delete b;
    return 0;
}
```

В этом примере `typeid` используется для проверки типа объекта, на который указывает `b`, а `dynamic_cast` используется для безопасного приведения указателя `b` к типу `Derived*`.


---

## Advanced

https://shaharmike.com/cpp/vtable-part1/
https://shaharmike.com/cpp/vtable-part2/
https://shaharmike.com/cpp/vtable-part3/
https://shaharmike.com/cpp/vtable-part4/

https://refspecs.linuxfoundation.org/cxxabi-1.86#vtable
https://hardwear.io/usa-2023/presentation/analyzing-decompiled-c++vtables-and-objects-in-GCC-binaries.pdf

https://nimrod.blog/posts/what-does-cpp-object-layout-look-like/#multiple-inheritance
https://nimrod.blog/posts/cpp-virtual-table-tables/


https://youtu.be/WdA2Lk601CI?si=kdO32ZLCNDjusrgG




