## Future

Трейт `std::future::Future` предоставляет единый интерфейс для всех видов асинхронных операций, которые могут завершиться в будущем, произведя некоторый результат в виде объекта ассоциированного типа `Output`. 

По своей концептуальной сути, каждый Future – это **конечный автомат** (`state machine`). Он неявно определяет состояния, через которые проходит асинхронное вычисление и логику переходов между ними. Однако внешний интерфейс `Future` абстрагирует эту внутреннюю сложность, сводя взаимодействие к одному ключевому методу – `poll(...)`, который управляет переходами между состояниями этого автомата.

``` Rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

Метод `poll()` вызывается для проверки состояния `Future`. Он возвращает состояние `Poll::Ready(T)`, если асинхронная операция завершена, в этом случае `Output = T`. В случае, если операция ещё не завершена, возвращается состояние `Poll::Pending`.

`Context<'_>` в данном случае является структурой, которая предоставляет собой контекст выполнения. Самым важным в нём является ссылка на `Waker`. `Waker` - это механизм, с помощью которого `Future`, находящийся в состоянии `Pending`, может сигнализировать исполнителю (`Executor`), что данная асинхронная операция, представленная этим `Future`, возможно, готова к следующей проверке состояния через метод `poll`. Например, потому что пришли данные из сокета.


**Критически важный контракт poll:** Если `Future` возвращает `Poll::Pending`, он **обязан** зарегистрировать `Waker`, полученный из `Context`, таким образом, чтобы этот `Waker` был вызван (`waker.wake()`) позже, когда `Future` потенциально сможет продвинуться дальше (например, когда операционная система уведомит о поступлении данных в сокет). Если `Future` вернет `Poll::Pending` без организации будущего вызова `Waker::wake()`, он рискует "заснуть" навсегда, так как у исполнителя не будет оснований опрашивать его снова.

Параметр `Pin<&mut self>` будет разобран позже, так как является достаточно сложным в понимании. 


И, наконец, одно из самых фундаментальных свойств `Future`: **ленивость (laziness)**. В отличие от некоторых других моделей асинхронности (например, `Promises` в `JavaScript`, которые часто начинают выполняться сразу после создания), `Future` в `Rust` не делает абсолютно ничего сам по себе. Простое создание экземпляра `Future` (например, вызов `async fn`) не инициирует никаких вычислений. Реальное выполнение начинается только в тот момент, когда **исполнитель (`Executor`)** впервые вызывает метод `poll` для этого `Future`. 

Исполнитель — это сущность (обычно часть асинхронного рантайма), которая активно управляет жизненным циклом `Future`: принимает их на выполнение, планирует вызовы `poll` для готовых `Future`, обрабатывает сигналы от `Waker` и, в конечном итоге, "доводит" `Future` до состояния `Poll::Ready`. Без исполнителя `Future` останется просто пассивным описанием вычисления.
### state machine

Возникает фундаментальный вопрос: если метод `poll` возвращает `poll::Pending`, текущий поток выполнения уступает управление, и стековый фрейм соответствующей функции уходит со стека вызовов. Как же тогда `Future` сможет возобновить работу с того же места и с тем же значением ресурса, используемого в этой функции при следующем вызове `poll`?

Ответ кроется в том, как компилятор преобразует `async fn / async{}`. Компилятор трансформирует тело асинхронной функции в структуру, которая сама имплементирует трейт `Future`. Эта сгенерированная структура и есть тот самый конечный автомат. Все локальные переменные внутри асинхронного контекста, значения которых должны быть сохранены между точками `.await` становятся полями данной сгенерированной структуры. Данная структура также содержит `enum`, который отслеживает текущее состоянии конечного автомата - то есть, на какой точке `.await` выполнение было приостановлено в прошлый раз. В случае, если внутри `async fn` происходит ожидание `.await` для вложенной `Future`, она также сохраняется как поле в сгенерированной структуре на время своего выполнения.

В качестве примера рассмотрим простую асинхронную функцию:

``` Rust
async fn example_task(init_value: u32) -> u32 {
	println!("Task started with {}", init_value);
	let intermediate = some_async_op(init_value).await; // #1 await
	println!("Intermediate value: {}", intermediate);
	let final_val = another_async_op(intermediate).await; // #2 await
	final_val + 10
}
```


Компилятор может сгенерировать примерно такую структуру (концептуальное представление для понимания):


``` Rust
struct ExampleTaskFuture {
    // Fields for storing state (local variables)
    initial_value: u32,       // Preserved until first await
    intermediate: Option<u32>, // Preserved between await #1 and #2
    // final_val doesn't need to be stored as a field, used directly for return

    // Fields for storing nested Futures
    state_some_async_op: Option<impl Future<Output = u32>>,
    state_another_async_op: Option<impl Future<Output = u32>>,

    // Current state machine state
    current_state: TaskState,
}

enum TaskState {
    Start,
    WaitingOnSomeOp,
    WaitingOnAnotherOp,
    Done,
}

impl Future for ExampleTaskFuture {
    type Output = u32;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 'unsafe' is used to demonstrate field access through Pin
        let this = unsafe { self.as_mut().get_unchecked_mut() };

        loop { // Allows transitioning between states in a single poll call
            match this.current_state {
                TaskState::Start => {
                    println!("Task started with {}", this.initial_value);
                    // Create first Future and store it
                    this.state_some_async_op = Some(some_async_op(this.initial_value));
                    this.current_state = TaskState::WaitingOnSomeOp;
                    // Don't return, continue polling the nested Future
                }
                TaskState::WaitingOnSomeOp => {
                    // Get Future for polling
                    let fut = unsafe { Pin::new_unchecked(this.state_some_async_op.as_mut().unwrap()) };
                    match fut.poll(cx) {
                        Poll::Ready(value) => {
                            // First Future completed, store result
                            this.intermediate = Some(value);
                            this.state_some_async_op = None; // Free Future memory
                            println!("Intermediate value: {}", value);

                            // Create second Future
                            this.state_another_async_op = Some(another_async_op(value));
                            this.current_state = TaskState::WaitingOnAnotherOp;
                            // Continue polling second Future
                        }
                        Poll::Pending => {
                            return Poll::Pending; // Operation not ready, yield
                        }
                    }
                }
                TaskState::WaitingOnAnotherOp => {
                     let fut = unsafe { Pin::new_unchecked(this.state_another_async_op.as_mut().unwrap()) };
                     match fut.poll(cx) {
                         Poll::Ready(final_val) => {
                             // Second Future completed
                             this.state_another_async_op = None;
                             this.current_state = TaskState::Done;
                             // Calculate and return final result
                             return Poll::Ready(final_val + 10);
                         }
                         Poll::Pending => {
                             return Poll::Pending; // Operation not ready
                         }
                     }
                }
                 TaskState::Done => {
                     panic!("Polled a completed future");
                 }
            }
        }
    }
}
```

Таким образом, вся необходимая информация для возобновления задачи хранится **внутри самого объекта `Future`**. Когда исполнитель вызывает `poll`, метод смотрит на `current_state`, использует сохраненные значения из полей структуры, выполняет необходимую логику (например, вызывает `poll` для вложенного `Future`), обновляет состояние и поля при необходимости, и либо возвращает `Poll::Ready`, либо `Poll::Pending`.


---

## `async/await`

Несложно догадаться, что прямая имплементация `Future` для наших операций - совсем нетривиальная задача, так как она требует явного управления состояниями, обработки `Waker`. Все это делает ручное определение `Future` для наших типов многословным и трудным для чтения и поддержки. Нужен был способ писать асинхронный код, который выглядел бы почти так же просто, как синхронный код, но при этом выполнялся бы неблокирующим образом. Именно эту проблему и решают ключевые слова в языке `async` и `await`.

`async/await` – это **синтаксический сахар поверх модели `Future`, предоставляемый компилятором.

### `async`

Ключевое слово `async` используется для трансформации функции или блока кода таким образом, чтобы они возвращали значение, имплементирующее трейт `Future`, вместо того, чтобы классически выполнять код немедленно и возвращать конечный результат.

#### `async fn`

Объявление функции с ключевым словом `async` изменяет её сигнатуру. Например в случае `async fn my_operation() -> Result<u32, Error>`, вместо того, чтобы возвращать явно указанный тип `Result<u32, Error>`, такая функция вернёт анонимный тип, имплементирующий `Future`, инкапсулировав возвращаемое значение исходной функции в ассоциированный тип `Output`: `impl Future<Output = Result<u32, Error>>`.

Рассмотрим практический пример асинхронной функции:

``` Rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

async fn fetch_data_from_server() -> Result<(), std::io::Error> {
    println!("Connecting to example.com:80...");
    
    // connecting to the server
    let mut stream = match TcpStream::connect("example.com:80").await {
        Ok(stream) => {
            println!("Connected successfully!");
            stream
        },
        Err(e) => {
            println!("Connection failed: {}", e);
            return Err(e);
        }
    };
    
    if let Err(e) = stream.write_all(b"GET / HTTP/1.0\r\n\r\n").await {
        println!("Write failed: {}", e);
        return Err(e);
    }
    println!("Request sent");
    
    let mut buf = [0; 1024];
    match stream.read(&mut buf).await {
        Ok(n) => {
            println!("Received {} bytes:\n{}", n, String::from_utf8_lossy(&buf[..n]));
            Ok(())
        },
        Err(e) => {
            println!("Read failed: {}", e);
            Err(e)
        }
    }
}

#[tokio::main]
async fn main() {
    let _ = fetch_data_from_server().await;
}
```

Подробный desugaring, реализуемый компилятором можно увидеть, если транслировать код в `MIR` [тык](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=29f66cfe711cda0217bd798ec33306cb).

#### `async` blocks

`async` блок создаёт `Future` непосредственно из аннотированного блока кода. Он полезен для создания `Future` "на месте", часто для передачи в функции типа `tokio::spawn` или комбинаторы `Future` (например, `futures::future::join`). Также `async` блоки могут захватывать объекты из окружающей области видимости, аналогично замыканиям.

``` Rust
async fn run_concurrently() {
    let name = "World".to_string();

	// async block
    let future_block = async {
        println!("Async block started.");
        tokio::time::sleep(std::time::Duration::from_millis(5)).await;
        println!("Hello from async block!");
    };

	// async move 
    let future_move_block = async move {
        println!("Async move block started for {}.", name);
        tokio::time::sleep(std::time::Duration::from_millis(5)).await;
        format!("Greetings, {}!", name)
    };


    let (_, result) = futures::join!(future_block, future_move_block);
    println!("Result from move block: {}", result);
}
```


---
### `.await`

Мы установили, что ключевое слово `async` создает Future, инкапсулирующий асинхронную операцию. Но как нам получить результат этой операции внутри другой асинхронной функции, не прибегая к низкоуровневому `poll` и ручному управлению состоянием? Как превратить обещание будущего результата в конкретное значение, когда оно станет доступно, не останавливая при этом весь поток выполнения? Ответ кроется в операторе `.await`.

Оператор `.await` — это не просто синтаксическая конструкция, это **точка взаимодействия** между асинхронной задачей и исполнителем (`Executor`).

Итак, что же технически происходит, когда исполнение асинхронной функции достигает точки `expression.await`? На самом деле, компилятор преобразует это в относительно сложную, но четко определённую последовательность действий, взаимодействуя с механизмом `Future::poll` и исполнителем.

Сначала вычисляется выражение слева от `.await` (expression). Результатом должно быть значение, реализующее `Future` – назовем его `inner_future`.
    
Немедленно вызывается метод `inner_future.poll(cx)`. Контекст `cx` (содержащий Waker, необходимый для пробуждения) передаётся из вызова `poll` родительской асинхронной функции (той, внутри которой находится данный `.await`). Здесь пути расходятся:
    
**Случай 1: `Poll::Ready(value)`**. `inner_future` смогла завершиться немедленно во время вызова `poll` и вернула готовое значение `value`. Это может произойти, если асинхронная операция была очень быстрой или уже была завершена ранее. В этом случае выражение `inner_future.await` преобразуется в `value`. Выполнение родительской async функции **продолжается немедленно** со следующей инструкции. В этом сценарии `.await` ведет себя почти как обычный вызов функции – **приостановки не происходит**.
            
**Случай 2: `Poll::Pending`**. `inner_future` не смогла завершиться на момент вызова `poll`. Она, возможно, выполнила часть работы (например, отправила запрос по сети), но теперь должна ожидать внешнего события (например, получения ответа). Прежде чем вернуть `Poll::Pending`, реализация `inner_future.poll()` **обязана** сохранить или зарегистрировать предоставленный `Waker` (из `cx`). Этот `Waker` уникально идентифицирует родительскую задачу и содержит механизм для её "пробуждения" (сигнала исполнителю, что задача снова может быть готова к выполнению). `Waker` связывает внешнее событие, которого ожидает `inner_future`, с родительской задачей, ожидающей результата `inner_future`.
            
В этот самый момент родительская `async` функция/блок **приостанавливается**. Её выполнение останавливается именно в точке `.await`. Компилятор гарантирует, что все локальные переменные и другое состояние родительской функции, необходимые для её возобновления позже, **сохраняются**.
            
Вызов `poll` родительской функции немедленно возвращает `Poll::Pending` исполнителю. Это сигнал: "Эта задача сейчас не может продвинуться, попробуйте позже".
            

Крайне важно: поток, в котором выполнялся `poll`, **не блокируется**. Исполнитель теперь свободен использовать этот поток для опроса других готовых к выполнению асинхронных задач или для взаимодействия с системным механизмом отслеживания событий ввода-вывода (реактором).
            
Значительно позже, когда произойдет внешнее событие, которого ожидала `inner_future` (например, операционная система уведомит о поступлении данных в сокет), механизм, отслеживающий это событие (реактор рантайма), вызовет метод `wake()` на сохраненном `Waker`. Вызов `wake()` уведомляет исполнителя, что родительская задача (которая была приостановлена на `.await`) теперь, вероятно, готова продолжить выполнение.
            
После этого, исполнитель помещает родительскую задачу в очередь готовых к выполнению. При следующем удобном случае исполнитель снова вызовет `poll` для родительской задачи. Её выполнение **возобновится** точно с того места, где она была приостановлена – то есть, она снова попытается выполнить `inner_future.await`, повторяя шаги 2 и 3.
            
Этот цикл опроса (`poll`), приостановки (`Pending`), пробуждения (`wake`) и возобновления (`poll` снова) продолжается до тех пор, пока `inner_future.poll()` наконец не вернет `Poll::Ready(value)`.



``` Rust
async fn example() {
    // 1. Calling get_string_async() creates a Future object
    //    - This DOES NOT start execution yet (Futures are lazy)
    //    - The future_obj is of type impl Future<Output = String>
    let future_obj = get_string_async();

    // 2. The .await point - where the magic happens:
    //    - The compiler generates code that calls future_obj.poll(cx)
    //    - This is the suspension/resumption point
    let result: String = future_obj.await;
    // ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    // | When poll() returns Poll::Ready("Hello..."):
    // |   - 'result' gets assigned "Hello..."
    // |   - Execution immediately continues to println!
    // |
    // | When poll() returns Poll::Pending:
    // |   - The entire example() function PAUSES HERE
    // |   - Its state (variables, execution point) is saved
    // |   - Control returns to the executor (tokio/async-std/etc)
    // |   - Later... when the Waker triggers...
    // |   - The executor calls poll() on example() again
    // |   - Execution RESUMES by retrying future_obj.await
    // |     (which means calling future_obj.poll(cx) again)

    // 3. This code only executes after the .await completes
    //    successfully (returns Poll::Ready)
    println!("Received: {}", result);
}

/// A simple async function that returns immediately
/// (Illustrates the Future state machine concept)
async fn get_string_async() -> String {
    // In this simplified example:
    // - There's no real async work (no I/O, no timers)
    // - Therefore it will likely return Poll::Ready immediately
    // 
    // For comparison, imagine if this contained:
    //   tokio::time::sleep(Duration::from_secs(1)).await;
    // Then:
    // - get_string_async would itself pause at its internal .await
    // - When example() first polls it, it would return Poll::Pending
    // - The Waker would trigger after ~1 second
    // - Then execution would resume
    
    "Hello, await!".to_string()
}
```

---

## `Parallellism and spawning`

Мы установили, что оператор `.await` приостанавливает выполнение текущей асинхронной задачи (её `Future`), позволяя исполнителю переключиться на другие задачи до тех пор, пока ожидаемая операция не сможет продолжить свое выполнение.

Однако, при последовательном выполнении `.await` для нескольких `future`, они будут выполняться строго одна за другой:

``` Rust
async fn sequential_operations {
	let result_1 = fetch_data("Source 1").await; // waiting for fetch_data for Source 1
	let result_2 = fetch_data("Source 2").await; // just after that we start and wait fetch_data for Source 2
	// ...
}
```

В приведенном коде операция для `Source 2` инициируется только после полного завершения операции для `Source 1`. Такой подход не всегда оптимален. Зачастую требуется инициировать несколько независимых асинхронных операций и позволить им выполняться **конкурентно** — то есть, прогрессировать одновременно, используя моменты ожидания одной операции для выполнения работы другой.

Важно отметить, что такая конкурентность может быть реализована с использованием **параллелизма**. Например, асинхронный рантайм, такой как `Tokio`, часто использует пул рабочих потоков (`thread pool`). В этом случае независимые асинхронные задачи могут выполняться действительно параллельно на разных ядрах процессора. Это позволяет масштабироваться при обработке большого количества одновременных операций (например, сетевых соединений), превосходя возможности асинхронности на одном потоке при высокой нагрузке. Именно для запуска таких конкурентных (и потенциально параллельных) задач и предназначена функция `tokio::spawn`.

Функция `tokio::spawn` имеет следующую сигнатуру:

``` Rust
pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
where 
	F: Future + Send + 'static,
	F::Output: Send + 'static,
{
	let task = Task::new(future);
    executor.schedule(task);
    JoinHandle { ... }
    // ...
}
```
Единственным аргументом функции является `Future`, которую мы хотим запустить конкурентно. Как правило, это результат вызова `async fn` или блока `async {}`.

Необходимо обратить внимание на **ограничения типов (`trait bounds`)**, накладываемые на передаваемый `Future` и его результат:

- **F: Future + Send + 'static**:
    
	- Future: Передаваемый объект должен реализовывать трейт `Future`.
        
    - `Send`: `Future` и все захваченные им данные должны быть безопасно передаваемыми между потоками (`Send`). Это требуется, поскольку планировщик рантайма может исполнять эту `Future` на любом из своих рабочих потоков.
        
    - `'static`: `Future` не должен содержать ссылок с временем жизни, меньшим чем `'static`. Это гарантирует безопасность памяти, так как порожденная задача может выполняться неопределенно долго и потенциально "пережить" контекст (например, функцию), в котором она была создана. Если бы `Future` содержал нестатические ссылки, они могли бы инвалидироваться. Обычно это ограничение удовлетворяется передачей владения в `Future` (через `async move {}`) или использованием потокобезопасных разделяемых указателей (`Arc`).
        
- **`F::Output: Send + 'static`**: Тип результата выполнения `Future` (`Output`) также должен удовлетворять ограничениям `Send` + `'static`, поскольку результат может быть передан через `JoinHandle` обратно в другой поток или сохранен на неопределенный срок после завершения задачи.


Рассмотрим концептуальный процесс работы `tokio::spawn`. Рантайм `Tokio` оборачивает предоставленный `future` во внутреннюю структуру данных, представляющую [задачу](https://docs.rs/tokio/latest/tokio/task/index.html) **(Task/green thread)**. Эта структура инкапсулирует сам `Future`, его текущее состояние выполнения и механизм пробуждения (`Waker`). Затем созданная задача передается **планировщику (`scheduler`)**. Планировщик - это компонент рантайма, ответственный за управление жизненным циклом всех активных асинхронных задач.

Задача помещается в очередь готовых к выполнению задач (`run queue`). **Рабочие потоки (`worker threads`)** рантайма непрерывно проверяют эту очередь. Когда рабочий поток освобождается, он извлекает готовую задачу и вызывает для ее `Future` метод `poll`. Этот процесс соответствует асинхронной модели, описанной в предыдущей главе: если `poll` возвращает `Poll::Pending`, задача приостанавливается до ее пробуждения через `Waker`; если `Poll::Ready`, задача считается завершенной. Благодаря наличию нескольких рабочих потоков и быстрому переключению между задачами, ожидающими ввода-вывода, достигается высокая степень конкурентности и потенциального параллелизма.


``` Rust
use reqwest::Client;
use tokio::time::{sleep, Duration};

async fn fetch_url(url: &'static str) -> Result<String, reqwest::Error> {
    sleep(Duration::from_millis(100)).await;

    Client::new()
        .get(url)
        .send()
        .await?
        .text()
        .await
}

#[tokio::main]
async fn main() {
    let urls: &'static [&'static str] = &[
        "https://httpbin.org/get",
        "https://httpbin.org/ip",
        "https://httpbin.org/user-agent",
        "https://httpbin.org/delay/1",
        "http://invalid-url-should-fail.tld",
    ];

    println!("Spawning tasks...");
    let start_time = std::time::Instant::now();

    let tasks: Vec<_> = urls.iter().map(|url| {
        let url_clone = *url;
        tokio::spawn(async move {
            match fetch_url(url_clone).await {
                Ok(body) => println!("{}: Received {} bytes", url_clone, body.len()),
                Err(e) => println!("{} failed: {}", url_clone, e),
            }
        })
    }).collect();

    println!("All tasks spawned. Waiting for completion...");

    for task in tasks {
        task.await.unwrap();
    }

    println!("All tasks finished in {:?}", start_time.elapsed());
}
```