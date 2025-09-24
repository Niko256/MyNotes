таблица символов - это структура данных, используемая компилятором для хранения и управления информацией о идентификаторах, которые являются частью исходного кода программы.

- таблица символов создаётся на этапе лексического анализа, и по окончанию компиляции является частью объектного файла.

- каждый идентификатор объекта из исходного кода ассоциируется с информацией, связанной с его объявлением: типом данных, именем, областью видимости, а также смещением (расположением в памяти процесса)

- Линковщик использует таблицу символов для определения, какие символы определены в каждом объектном файле.

	- Если в таблице символов объектного файла есть неопределенные символы (обозначенные как `*UND*`), линковщик ищет эти символы в других объектных файлах и библиотеках. 
		- Если он находит соответствующие определения, он связывает их вместе, в ином случае выдаёт ошибку линковки.

---
## Symbol table layout


- с помощью команды `objdump -x example.o` можно вывести в консоль всё содержимое указанного объектного файла

- с помощью [команды](https://linux.die.net/man/1/nm) `nm example.o` можно вывести только таблицу символов данного объектного файла


``` c++
// example.cpp
#include <iostream>

namespace MyNamespace {
    int globalVar = 42;

    void myFunction() {
        std::cout << "Hello from MyNamespace!" << std::endl;
    }

    class BaseClass {
    public:
        virtual void baseMethod() {
            std::cout << "Hello from BaseClass!" << std::endl;
        }
    };

    class DerivedClass : public BaseClass {
    public:
        void derivedMethod() {
            std::cout << "Hello from DerivedClass!" << std::endl;
        }
    };
}

int main() {
    MyNamespace::myFunction();
    MyNamespace::BaseClass baseObj;
    baseObj.baseMethod();
    MyNamespace::DerivedClass derivedObj;
    derivedObj.derivedMethod();
    return 0;
}
```


теперь с помощью команды `nm example.o --demangle` выведем таблицу символов данного объектного файла


``` c++
                 U _GLOBAL_OFFSET_TABLE_
0000000000000032 T main
                 U __stack_chk_fail
0000000000000000 T MyNamespace::myFunction()
0000000000000000 W MyNamespace::DerivedClass::derivedMethod()
0000000000000000 W MyNamespace::BaseClass::baseMethod()
0000000000000000 D MyNamespace::globalVar
                 U std::ostream::operator<<(std::ostream& (*)(std::ostream&))
0000000000000047 r std::__detail::__integer_to_chars_is_unsigned<unsigned int>
0000000000000048 r std::__detail::__integer_to_chars_is_unsigned<unsigned long>
0000000000000049 r std::__detail::__integer_to_chars_is_unsigned<unsigned long long>
                 U std::ios_base_library_init()
                 U std::cout
                 U std::basic_ostream<char, std::char_traits<char> >& std::endl<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&)
                 U std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)
0000000000000000 V typeinfo for MyNamespace::DerivedClass
0000000000000000 V typeinfo for MyNamespace::BaseClass
0000000000000000 V typeinfo name for MyNamespace::DerivedClass
0000000000000000 V typeinfo name for MyNamespace::BaseClass
                 U vtable for __cxxabiv1::__class_type_info
                 U vtable for __cxxabiv1::__si_class_type_info
0000000000000000 V vtable for MyNamespace::DerivedClass
0000000000000000 V vtable for MyNamespace::BaseClass

```


---
## [LLVM Implementation of Symbol tables](https://llvm.org/doxygen/ValueSymbolTable_8h_source.html)


[Object file layout](https://www.opennet.ru/soft/ruprog/coff.txt)
