**Файловый дескриптор** — это небольшое неотрицательное **целое число**, которое служит **абстрактным идентификатором (handle)** для ресурса ввода/вывода, открытого процессом и управляемого ядром операционной системы.


При этом файловые дескрипторы представляют далеко не только файлы на диске, а также множество других ресурсов:

- Директории
- Сетевые сокеты
- Каналы (`pipes`)
- Терминалы (`tty`)
- Механизмы `IPC`
- Специальные объекты ядра (`epoll instance`, `timerfd`, `signalfd`, `eventfd`, `etc...`)

То есть сила файловых дескрипторов в том, что они предоставляют **единый интерфейс** для работы с различными типами ресурсов ввода/вывода. Системные вызовы, такие как `read()`, `write()`, `close()`, `select()`, `poll()`, `epoll_ctl()`, оперируют именно файловыми дескрипторами, независимо от того, что за ними скрывается — файл, пайп или сетевой сокет.


**Каждый процесс** имеет **таблицу файловых дескрипторов** (`File Descriptor Table`), которая является частью контекста процесса.


``` c
struct files_struct {
    atomic_t count;           // reference counter
    struct fdtable* fdt;      // the main table
    // ...
};

struct fdtable {
    unsigned int max_fds;     // current size of the table
    struct file** fd;
    // ...
};

// for example:
FD Index  │ Referring to
──────────┼──────────────────────────────────────────
    0     │ struct file для /dev/pts/0 (terminal)
    1     │ struct file для /dev/pts/0 (terminal)
    2     │ struct file для /dev/pts/0 (terminal)
    3     │ struct file → struct socket (TCP-socket)
    4     │ struct file → pipe (reading)
    5     │ struct file → pipe (writing)
    6...  │ NULL (unused FD)
    ...   │
──────────┼──────────────────────────────────────────

struct file {
    mode_t f_mode;     // permissions (read/write)
    loff_t f_pos;      // current position in the file
    struct file_operations* f_op; // operations (read, write, poll)
    void* private_data;
};
```

При этом для сокетов `struct file->private_data` указывает на `struct socket`:

``` c
struct socket {
    struct sock *sk;  // real socket in the kernel
    // ...
};
```

