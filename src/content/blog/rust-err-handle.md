---
title: Преобразование типов ошибок в Rust
description: При использовании в Rust оператора '?' иногда хочется преобразовать тип ошибки
pubDate: 2026-1-22
---

Мне было не совсем очевидно, как преобразовывать тип возмращаемой ошибки в Rust и я решил в этом разобраться.

Начну с примера: у нас есть некая функция, которая вызывает внутри себя другие функции, возвращающие разный тип ошибок. Хотелось бы, что бы в каждом месте вызов не нужно было писать что то вида

```rust
let res = match fn_call(args) {
    Ok(res) => res,
    Err(e) => MyErr(e.kind)
}
```

А использовать стандартный оператор `?`:

```rust
#[derive(Debug)]
struct MyError {
    message: String
}

fn zero_err() -> Result<u8, MyError> {
    Err(MyError { message: "zero".to_string() })
}

fn io_err() -> Result<u8, io::Error> {
    Err(io::Error::new(io::ErrorKind::AddrInUse, "used"))
}

fn may_error_fn(errtype: u8) -> Result<u8, MyError> {
    let zero = zero_err()?;
    let ioe = io_err()?;
    if zero == ioe {
        return Ok(0);
    }
    Ok(errtype)
}
```

И это на удивление легко сделать!

```rust
impl From<io::Error> for MyError {
    fn from(value: io::Error) -> Self {
        MyError { message: value.to_string() } // Конечно, тут хорошо бы положить ошибку в Box и сделать у MyError поле reason, но это пример
    }
}
```

Вот и всё! Это не хак, такое поведение указанно в [официальной документации](https://doc.rust-lang.org/rust-by-example/std/result/question_mark.html), но почему то часто упускается и кажется не столь очевидным.