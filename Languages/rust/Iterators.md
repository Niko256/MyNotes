## basics

Итератор - это абстракция, которая представляет объект, указывающий на элемент какого-либо множества элементов.

- [Итератор является обобщением указателя.](https://stackoverflow.com/questions/2728190/how-are-iterators-and-pointers-related)
- Итератор предоставляет унифицированный доступ к элементу коллекции, не раскрывая её внутреннее устройство.


$\triangle$ Итераторы в Rust представлены через основной трейт [Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html)

``` Rust 
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    ... // other methods (76)
}
```

---
## types of iterating

Rust поддерживает 2 вида итераций:
- по ссылкам
- по владению

---
### Iterating by taking ownership

Итерация с передачей владения определяет поведение, при котором элементы коллекции перемещаются в функцию или блок кода, что убирает возможность дальнейшего обращения к элементам коллекции. 

Обычно этот вид итерации достигается за счёт метода `into_iter()`, который и забирает владение.

``` Rust
// example
fn main() {
	let v = vec![1, 2, 3];

	for i in v.into_iter() {
		printl!("{}", i);
	}
}
```

в данном примере дальнейшая попытка доступа к `v` вызовет ошибку, так как элементы вектора `v` были перемещёны методом `into_iter()` в цикл `for`.


$\triangle$ "прямая" итерация через цикл `for` является синтаксическим сахаром в случае итераторов. 

- компилятор будет вызывать метод `into_iter()` для передачи владения элементами.

``` Rust
let values = vec![1, 2, 3, 4, 5];

for x in values {
    println!("{x}");
}
```
``` Rust
// compiler will de-sugar it into:
let values = vec![1, 2, 3, 4, 5];
{
    let result = match IntoIterator::into_iter(values) {
        mut iter => loop {
            let next;
            match iter.next() {
                Some(val) => next = val,
                None => break,
            };
            let x = next;
            let () = { println!("{x}"); };
        },
    };
    result
}
```

---
### Iterating by references

Итерация через ссылки определяет заимствование элементов коллекции, что предоставляет возможность обращения и изменения элементов коллекции, а также позволяет избежать копирования элементов.

итерация по ссылкам определяется методами `iter()` и `iter_mut()`.

``` Rust
// example
let mut vec = vec![1, 2, 3, 4, 5];

// here item is an immutable reference
for item in vec.iter_mut() {
    *item *= 2;
}

// here &item is de-refed value
for &item in vec.iter() {
    println!("{}", item);
}
```

---
## [Lazyness of Iterators](https://doc.rust-lang.org/std/iter/index.html#laziness)

В Rust итераторы являются "ленивыми", что означает, что создание итератора не приводит к выполнению каких-либо действий до тех пор, пока не вызван метод `next()`.

``` Rust
// example
let v = vec![1, 2, 3, 4, 5];
v.iter().map(|x| println!("{x}"));
```

компилятор выдаст предупреждение:
``` Rust
warning: unused result that must be used: iterators are lazy and
do nothing unless consumed
```

исправить это можно следующим образом:

``` Rust
let v = vec![1, 2, 3, 4, 5];

v.iter().for_each(|x| println!("{x}"));
// or
for x in &v {
    println!("{x}");
}
```

---
## [Powerful methods with iterators](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect)

существует множество различных методов, которые связаны с итераторами.

- `into_iter` - преобразование коллекции в итератор. Этот метод является частью трейта `IntoIterator`. 

``` Rust
fn into_iter(self) -> Self::IntoIter
```
``` Rust
// example
let vec = vec![1, 2, 3];
let iter = vec.into_iter();
```
---
- `from_iter()` - создание коллекции из итератора. этот метод является частью трейта  `FromIterator`.

``` Rust
fn from_iter<T>(iter: T) -> Self
       where T: IntoIterator<Item = A>;
```
``` Rust 
//example
use std::iter::FromIterator;
let vec = Vec::from_iter(1..4);
```
---
- `enumerate()` - Возврат нового итератора, в виде пары - 
  (текущий счётчик итерации, соответствующий элемент итератора)
	- итеративный счётчик ограничен значением `usize::MAX`, в случае переполнения значения произойдёт паника.

``` Rust
fn enumerable(self) -> Enumerate<Self>
where Self: Sized,
```
``` Rust
//example
let a = ['a', 'b', 'c'];

let mut iter = a.iter().enumerate();

assert_eq!(iter.next(), Some((0, &'a')));
assert_eq!(iter.next(), Some((1, &'b')));
assert_eq!(iter.next(), Some((2, &'c')));
assert_eq!(iter.next(), None);
```

---
- `map()` - применение заданного замыкания к каждому элементу и возврат нового итератора.

``` Rust
fn map<B, F>(self, f: F) -> Map<Self, F>
where Self: Sized, F: FnMut(Self::Item) -> B,
```
``` Rust
//example
let vec = vec![1, 2, 3];
let mapped: Vec<i32> = vec.into_iter().map(|x| x * 2).collect();
```
---
- `for_each()` - применение заданного замыкания к каждому элементу, но без создания и возврата нового итератора. 
	- Прямая альтернатива циклу `for`, за исключением того, что `break` и `continue` для замыкания не валидны.
	- возвращает `unit`

``` Rust
fn for_each<F>(self, f: F)
where Self: Sized, F: FnMut(Self::Item),
```
``` Rust
//example
let vec = vec![1, 2, 3];
vec.iter().for_each(|x| println!("{}", x));
```
---
- `collect()` - сбор элементов итератора в коллекцию

``` Rust
fn collect<B>(self) -> B
where B: FromIterator<Self::Item>, Self: Sized,
```
``` Rust
//example
let chars = vec!['a', 'b', 'c'];
let s: String = chars.into_iter().collect();
println!("{}", s); // "abc"
```

---
- `fold()` - аккумуляция элементов итератора в одно значение с помощью замыкания.
	- Замыкание состоит из двух аргументов: начального состояния аккумулятора и элемента итератора.
	- `fold()` объединяет элементы в левоассоциативном порядке, обновляя аккумулятор на каждой итерации
		- Это означает, что операция применяется слева направо.
		- Для ассоциативных операторов, таких как `+`, порядок объединения элементов не важен.
		- Для неассоциативных операторов, таких как `-`, порядок будет влиять на конечный результат.

``` Rust
fn fold<B, F>(self, init: B, f: F) -> B
where
    F: FnMut(B, Self::Item) -> B,
```
``` Rust
// left-associative nature of fold()
//example
let numbers = [1, 2, 3, 4, 5];
let zero = "0".to_string();

let result = numbers.iter().fold(zero, |acc, &x| {
    format!("({acc} + {x})")
});

assert_eq!(result, "(((((0 + 1) + 2) + 3) + 4) + 5)");
```

---
- `reduce()` - аккумуляция элементов итератора с помощью замыкания, частный случай метода `fold()`. 
	- может использовать первый элемент итератора в качестве начального состояния, если тип аккумулятора и тип элементов совпадают.
	-  возвращает `None`, если итератор пуст.
	- замыкание состоит из двух аргументов: аккумулятора и элемента итератора

``` Rust
fn reduce<F>(self, f: F) -> Option<Self::Item>
where Self: Sized, F: FnMut(Self::Item, Self::Item) -> Self::Item
```
``` Rust 
//example
let vec = vec![1, 2, 3, 4];

// fold()
let sum = vec.iter().fold(0, |acc, x| acc + x);
println!("{}", sum); // 10

// reduce()
let sum = vec.iter().reduce(|acc, x| acc + x);
println!("{:?}", sum); // Some(10)
```

---
- `size_hint()` - Возврат кортежа вида `(lower_bound, upper_bound)`
	- Если верхняя граница неизвестна - метод возвращает `None`

``` Rust
fn size_hint(&self) -> (usize, Option<usize>)
```

- `size_hint()` не гарантирует точного количества элементов и его основное предназначение связано с оптимизацией
- неправильная реализация этого метода может привести к обращению к недействительной памяти.

``` Rust
let a = [1, 2, 3];
let mut iter = a.iter();

assert_eq!((3, Some(3)), iter.size_hint());
let _ = iter.next();
assert_eq!((2, Some(2)), iter.size_hint());
```

---

- `nth()` - возврат n-го элемента итератора.
	- возвращает `None`, если n-й элемент больше или равен длине итератора.
	- повторная итерация по элементам и обратная итерация не определены

```Rust
fn nth(&mut self, n: usize) -> Option<Self::Item>
```
``` Rust
//example
let vec = vec![1, 2, 3, 4, 5];
let third_element = vec.iter().nth(2);
println!("{:?}", third_element); // Some(3)
```

---
- `position()` - поиск индекса первого элемента итератора, удовлетворяющего заданному предикату.
	- возвращает `None`, если каждый элемент итератора не удовлетворяет заданному предикату.

``` Rust
fn position<P>(&mut self, predicate: P) -> Option<usize>
where Self: Sized, P: FnMut(Self::Item) -> bool,
```
``` Rust
//example
let vec = vec![1, 2, 3];
let pos = vec.into_iter().position(|x| x == 2);
```

---
- `max()` и `min()` - поиск максимального и минимальных элементов итератора соответственно.

``` Rust
fn max(self) -> Option<Self::Item>
where Self: Sized, Self::Item: Ord,

fn min(self) -> Option<Self::Item>
where Self: Sized, Self::Item: Ord,
```
``` Rust
//example
let a = [1, 2, 3];
let b: Vec<u32> = Vec::new();

assert_eq!(a.iter().max(), Some(&3));
assert_eq!(b.iter().max(), None);
assert_eq!(a.iter().min(), Some(&1));
assert_eq!(b.iter().min(), None);
```

---

- `filter()` - фильтрация элементов итератора по заданному предикату.
	- возвращает итератор, который содержит содержит только те элементы, которые удовлетворяют предикату.

``` Rust
fn filter<P>(self, predicate: P) -> Filter<Self, P>
where Self: Sized, P: FnMut(&Self::Item) -> bool,
```
``` Rust
//example
let a = [0, 1, 2];

let mut iter = a.iter().filter(|x| **x > 1); // need two *s!

assert_eq!(iter.next(), Some(&2));
assert_eq!(iter.next(), None);
```

---
- `zip()` - объединение двух итераторов в единый итератор пар (a, b)
	- a - элемент первого итератора, b - элемент второго итератора.

``` Rust
fn zip<U>(self, other: U) -> Zip<Self, <U as IntoIterator>::IntoIter>)
where Self: Sized, U: IntoIterator,
```
``` Rust
//example
let a1 = [1, 2, 3];
let a2 = [4, 5, 6];

let mut iter = a1.iter().zip(a2.iter());

assert_eq!(iter.next(), Some((&1, &4)));
assert_eq!(iter.next(), Some((&2, &5)));
assert_eq!(iter.next(), Some((&3, &6)));
assert_eq!(iter.next(), None);
```

---
- `chain()` - Объединение двух итераторов в один единый, путём последовательного возврата элементов из первого итератора, а затем элементов из второго итератора.

``` Rust
fn chain<U>(self, other: U) -> Chain<Self, <U as IntoIterator>::IntoIter>
where Self: Sized, U: IntoIterator<Item = Self::Item>,
```
``` Rust
//example
let a1 = [1, 2, 3];
let a2 = [4, 5, 6];

let mut iter = a1.iter().chain(a2.iter());

assert_eq!(iter.next(), Some(&1));
assert_eq!(iter.next(), Some(&2));
assert_eq!(iter.next(), Some(&3));
assert_eq!(iter.next(), Some(&4));
assert_eq!(iter.next(), Some(&5));
assert_eq!(iter.next(), Some(&6));
assert_eq!(iter.next(), None);
```

---
- `take()` - Возврат первых n элементов исходного итератора.

``` Rust
fn take(self, n: usize) -> Take<Self>
where Self: Sized,
```
``` Rust
//example
let vec = vec![1, 2, 3];
let taken: Vec<i32> = vec.into_iter().take(2).collect();
```

---
- `partition()` - Разделение элементов итератора на две коллекции по заданному предикату.

``` Rust
fn partition<B, F>(self, f: F) -> (B, B)
where Self: Sized, B: Default + Extend<Self::Item>, F: FnMut(&Self::Item) -> bool
```
``` Rust
//example
let a = [1, 2, 3];

let (even, odd): (Vec<_>, Vec<_>) = a
    .into_iter()
    .partition(|n| n % 2 == 0);

assert_eq!(even, vec![2]);
assert_eq!(odd, vec![1, 3]);
```

---
- `all(), any()` - Проверка элементов итератора на заданный предикат
	- `all()` проверяет каждый элемент итератора на соответствие предикату
	- `any()` проверяет соответствие предикату какого-либо из элементов итератора.

``` Rust
fn all<F>(&mut self, f: F) -> bool
where Self: Sized, F: FnMut(Self::Item) -> bool,

fn any<F>(&mut self, f: F) -> bool
where Self: Sized, F: FnMut(Self::Item) -> bool,
```
``` Rust
// example
let a = [1, 2, 3];

assert!(a.iter().any(|&x| x > 0));

assert!(!a.iter().any(|&x| x > 5));
```

---

## Advanced

[1](https://blog.jetbrains.com/rust/2024/03/12/rust-iterators-beyond-the-basics-part-ii-key-aspects/)

[2](https://blog.jetbrains.com/rust/2024/03/12/rust-iterators-beyond-the-basics-part-iii-tips-and-tricks/)
