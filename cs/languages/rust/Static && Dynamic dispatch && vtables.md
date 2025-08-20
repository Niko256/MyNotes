## [Dynamic dispatch](https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/)

Динамическая диспатчеризация - это механизм, который позволяет вызывать методы для trait-объектов в процессе run-time. 


``` Rust
trait Draw {
    fn draw(&self);
}

struct Circle {
    radius: f64,
}

impl Draw for Circle {
    fn draw(&self) {
        println!("Drawing a circle with radius {}", self.radius);
    }
}

struct Square {
    side: f64,
}

impl Draw for Square {
    fn draw(&self) {
        println!("Drawing a square with side {}", self.side);
    }
}


fn main() {
    let circle = Circle { radius: 5.0 };
    let square = Square { side: 10.0 };

    let shapes: Vec<Box<dyn Draw>> = vec![
        Box::new(circle),
        Box::new(square),
    ];

    for shape in shapes {
        shape.draw();
    }
}
```


`dyn Draw` - это trait-объект (тип), который указывает на любой объект, реализующий trait `Draw`.

---

### [Trait object layout](https://alexeden.github.io/learning-rust/programming_rust/11_traits_and_generics.html#:~:text=In%20memory%2C%20a%20trait%20object,called%20a%20virtual%20table%20(vtable))


- `Trait object` представлен в виде двух указателей 
  (wide pointer):

	- `data pointer`. указатель на данные объекта, реализующего trait
		
	- указатель на `vtable (vptr)`, которая содержит указатели на реализации методов для конкретного типа, реализующего данный trait
	
``` Rust
pub struct TraitObject {
	pub data: *mut (),
	pub vtable: *mut (),
}
```

---

## Vtables

`Vtable` (`Virtual Method Table`) — это структура данных, используемая в языках программирования для поддержки динамической диспетчеризации. 


Vtable состоит из:
- указателей на реализации методов для конкретного типа
- `Drop::drop`, `drop glue`
- [size, alignment](https://stackoverflow.com/questions/52011247/why-do-trait-object-vtables-contain-size-and-alignment)
- ссылок на остальные основные методы `(copy(), clone(), ...)`

Это позволяет вызывать методы во время выполнения программы, основываясь на типе объекта.

---
### Свойства и принцип работы

- `vtables` создаются на этапе компиляции [в сегменте](https://alschwalm.com/blog/static/2017/01/24/reversing-c-virtual-functions-part-2-2/) `data`
	- при создании объекта, тип, которого реализует данный trait, компилятор создаёт отдельную `vtable`

- в процессе runtime при вызове метода происходит поиск правильной реализации метода для данного типа

- происходит динамический вызов подходящего метода


рассмотрим пример, приведённый выше:

``` Rust
impl Draw for Circle {
    fn draw(&self) {
        println!("Drawing a circle with radius {}", self.radius);
    }
}

fn main() {
    let circle: Box<dyn Draw> = Box::new(Circle { radius: 3.0 });
    let square: Box<dyn Draw> = Box::new(Square { side: 4.0 });

    let shapes: Vec<Box<dyn Draw>> = vec![circle, square];

    for shape in shapes {
        shape.draw();
    }
}
```


- Компилятор создаёт `vtable` для объекта типа `Circle`, которая содержит указатель на метод `draw` для `Circle`.

- Компилятор создает `vtable` для объекта типа `Square`, которая содержит указатель на метод `draw` для `Square`.

- При создании `Box<dyn Draw>` компилятор создаёт структуру, состоящую из двух указателей, один из которых указывает на область памяти на куче для хранения этого trait-объекта, а второй - на соответствующую `vtable`.

- При вызове `shape.draw()` компилятор использует указатель на соответствующую `vtable` для поиска *правильного* метода `draw`, а затем вызовет его. 


---

## Static dispatch

**Статическая диспетчеризация** — это механизм, при котором компилятор Rust определяет во время компиляции, какой именно метод или функция будет вызвана. 

``` Rust
trait Foo {
    fn method(&self) -> String;
}

impl Foo for u8 {
    fn method(&self) -> String { format!("u8: {}", *self)}
}

impl Foo for String {
    fn method(&self) -> String { format!("string: {}", *self) }
}

fn do_something<T: Foo>(x: T) {
    x.method();
}

fn main() {
    let x = 5u8;
    let y = "Hello".to_string();

    do_something(x);
    do_something(y);
}
```

Rust использует *Мономорфизацию* для реализации статической диспатчеризации.

Это означает, что компилятор создаст обобщённую версию функции `do_something` для `u8` и `String`.

Затем компилятор заменит вызовы обобщённой функции на вызовы специализированных функций `do_something_u8` и `do_something_String`, которые он создаст.

Другими словами, компилятор сделает следующее:

``` Rust
fn do_something_u8(x: u8) {
    x.method();
}

fn do_something_string(x: String) {
    x.method();
}

fn main() {
    let x = 5u8;
    let y = "Hello".to_string();

    do_something_u8(x);
    do_something_string(y);
}
```



- При статической диспетчеризации компилятор знает, какой именно метод будет вызван.
	- Это позволяет компилятору выполнять оптимизации, такие как встраивание (inlining) методов, что может привести к более высокой производительности.
    
		- Встраивание методов означает, что код метода вставляется непосредственно в место вызова, вместо того чтобы делать отдельный вызов функции.
			- Это может уменьшить накладные расходы на вызов функции и улучшить производительность.
    

- **Увеличение размера кода (code bloat)**: Поскольку компилятор создает отдельные копии функции для каждого конкретного типа, это может привести к увеличению размера конечного бинарного файла.
