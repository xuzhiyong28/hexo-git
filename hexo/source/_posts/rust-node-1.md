---
title: Rust学习笔记
tags:
  - rust
categories: rust
description : Rust学习笔记
date: 2020-08-14
---

<!--more-->

### 笔记

***自定义类型如何打印输出？***

自定义的类型如果要进行:?打印输出，需要实现`fmt::Debug`的`trait`。也可以在类型上加`#[derive(Debug)]`

```rust
fn main(){
    #[derive(Debug)]
	struct UnPrintable(i32);
	println!("{:?}", DebugPrintable(13));
    #[derive(Debug)]
    struct Person<'a> {
        name: &'a str,
        age: u8
    }
    let name = "Peter";
    let age = 27;
    let peter = Person { name, age };
    // 美化打印
    println!("{:#?}", peter);
}
```

***消除变量未使用告警***

```rust
let immutable_binding = 1;  //定义了这个变量，但是没有使用，编译时会告警
let _immutable_binding = 1; //但是如果变量名前面加_ 则不会告警
```

***Rust不提供原生类型的隐式转换，如果要转换可以通过as显式转换***

```rust
let decimal = 65.4321_f32;
// 错误！不提供隐式转换
let integer: u8 = decimal;
// 改正 ^ 注释掉这一行
// 可以显式转换
let integer = decimal as u8;
```

***结构体的类型转换***

- From 和 Into
- TryFrom 和 TryInto 类似于From和Into，只是这组返回的是Result类型
- ToString 和 FromStr 可以把任何类型转换成string

```rust
use std::convert::From;
#[derive(Debug)]
struct Number {
    value: i32,
}
// 实现了From<i32>表示i32类型可以通过into()方法转换成Number类型
// i32转Number let num: Number = 5.into();
impl From<i32> for Number {
    fn from(item: i32) -> Self {
        Number { value: item }
    }
}
fn main(){
    let int: i32 = 5;
    let num: Number = int.into();
    println!("My number is {:?}", num);
}
```

```rust
use std::convert::TryFrom;
use std::convert::TryInto;
#[derive(Debug, PartialEq)]
struct EvenNumber(i32);
impl TryFrom<i32> for EvenNumber {
    type Error = ();
    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value % 2 == 0 {
            Ok(EvenNumber(value))
        } else {
            Err(())
        }
    }
}
fn main(){
    let result: Result<EvenNumber, ()> = 8i32.try_into();
    assert_eq!(result, Ok(EvenNumber(8)));
    let result: Result<EvenNumber, ()> = 5i32.try_into();
    assert_eq!(result, Err(()));
}
```

***Rust中几种迭代方法***

- iter() , 在每次迭代中借用集合中的一个元素。这样集合本身不会被改变，循环之后仍 可以使用。
- into_iter() , 会消耗集合。在每次迭代中，集合中的数据本身会被提供。一旦集合被消 耗了，之后就无法再使用了，因为它已经在循环中被 “移除”（move）了。
- iter_mut() , 可变地（mutably）借用集合中的每个元素，从而允许集合被就地修改。

***Rust闭包move***

move的作用是`将所引用的变量的所有权转移至闭包内`。不带move则不会转移所有权。

```rust
fn main() {
    struct Point {
        x: i64,
        y: i64,
    }
    if true {
        let mut p = Point { x: 25, y: 25 };
        println!("p address: {:p}", &p);
        (|| {
            println!("不带move闭包，p address: {:p}", &p);
        })();
        println!("p address: {:p}", &p);
        println!("------------------------------");
    }
    if true {
        let mut p = Point { x: 25, y: 25 };
        println!("p address: {:p}", &p);
        (move || {
            println!("带move闭包，p address: {:p}", &p);
        })();
        //println!("p address: {:p}", &p);//error[E0382]: borrow of moved value: `p`
    }
}
```

***Rust条件编译***

条件编译可能通过两种不同的操作符实现：

- cfg 属性：在属性位置中使用 #[cfg(...)]
- cfg! 宏：在布尔表达式中使用 cfg!(...)

`#[cfg(xxx)]` 和 `#[cfg(not(xxx))]`必须是一组，不然编译报错。也挺好理解的，既然是条件编译，如果没有所有条件都满足就会编译报错。

```rust
// 这个函数仅当目标系统是 Linux 的时候才会编译
#[cfg(target_os = "linux")]
fn are_you_on_linux() {
    println!("You are running linux!")
}
// 而这个函数仅当目标系统不是Linux时才会编译
#[cfg(not(target_os = "linux"))]
fn are_you_on_linux() {
    println!("You are *not* running linux!")
}
pub fn demo13(){
    are_you_on_linux();
}
```

`target_os`是系统提供的，如果我们要用自定义的条件编译，需要通过`--cfg xxx`指定。

```rust
#[cfg(some_condition)]
fn conditional_function() {
    println!("condition met!")
}
#[cfg(not(some_condition))]
fn conditional_function() {
    println!("condition met not !")
}
fn main() {
    conditional_function();
}
//运行 rustc --cfg some_condition xxx.rs && ./custom
```

***Rust特征(trait)中泛型参数和关联类型区别***

trait 中的泛型与关联类型，有如下区别：

- 如果 trait 中包含泛型参数，那么，可以对同一个目标类型，多次 impl 此 trait，每次提供不同的泛型参数。而关联类型方式只允许对目标类型实现一次。
- 如果 trait 中包含泛型参数，那么在具体方法调用的时候，必须加以类型标注以明确使用的是哪一个具体的实现。而关联类型方式具体调用时不需要标注类型（因为不存在模棱两可的情况）。

```rust
pub trait Converter<T> {
    fn convert(&self) -> T;
}
struct MyInt;
impl Converter<i32> for MyInt {
    fn convert(&self) -> i32 {
        42
    }
}
fn main(){
    let my_int = MyInt;
    let output: i32 = my_int.convert();
    println!("output is: {}", output);
    
    let output: f32 = my_int.convert(); //报错，因为必须为MyInt也实现Converter<f32>的trait才行
    println!("output is: {}", output);
}
```

***如何返回trait类型***

下面的代码编译错误，原因是 ：所有函数都必须返回一个具体类型。与其他语言不同，如果你有个像 `Animal` 那样的的 trait，则不能编写返回 `Animal` 的函数，因为其不同的实现将需要不同的内存量。

解决方法是使用 `Box + dyn`

```rust
struct Sheep {}
struct Cow {}
trait Animal {
    fn noise(&self) -> &'static str;
}
impl Animal for Sheep {
    fn noise(&self) -> &'static str {
            "baaaaah!"
    }
}
impl Animal for Cow {
    fn noise(&self) -> &'static str {
        "moooooo!"
    }
}
fn random_animal(random_number: f64) -> impl Animal {
    if random_number < 0.5 {
    	Sheep {}
    }else{
	    Cow {}
    }
}
fn random_animal_2(random_number: f64) -> Box<dyn Animal> {
    if random_number < 0.5 {
        Box::new(Sheep {})
    }else {
        Box::new(Cow {})
    }
}
fn main {
    let random_number = 0.234;
    let animal = random_animal_2(random_number);
    println!("You've randomly chosen an animal, and it says {}", animal.noise());
}
```

***Option<T>枚举类型 和 unwrap方法***

- `Some(T)` : 找到一个属于 `T` 类型的元素
- `None` : 找不到相应元素

```rust
fn give_commoner(gift: Option<&str>) {
    match gift {
        Some("snake") => println!("Yuck! I'm throwing that snake in a fire."),
        Some(inner)   => println!("{}? How nice.", inner),
        None          => println!("No gift? Oh well."),
    }
}
fn give_princess(gift: Option<&str>) {
    // `unwrap` 在接收到 `None` 时将返回 `panic`。
    let inside = gift.unwarp();
    if inside == "snake" { 
        panic!("AAAaaaaa!!!!");
    }
    println!("I love {}s!!!!!", inside);
}
fn main(){
	let food  = Some("chicken");
    let snake = Some("snake");
    let void  = None;
    give_commoner(food);
    give_commoner(snake);
    give_commoner(void);

    let bird = Some("robin");
    let nothing = None;
    give_princess(bird);
    give_princess(nothing);
}
```

使用`?`解开`Option`

- 如果 `current_age` 是 `None`，这将返回 `None`。
- 如果 `current_age` 是 `Some`，内部的 `u8` 将赋值给 `next_age`。

```rust
fn next_birthday(current_age: Option<u8>) -> Option<String> {
    let next_age: u8 = current_age?;
    Some(format!("Next year I will be {}", next_age))
}
```

### 附录

| 代码                                       | 说明                                                 |
| ------------------------------------------ | ---------------------------------------------------- |
| #[derive(Debug)]                           | 自动实现`Debug`特征                                  |
| #![allow(dead_code)]                       | 该属性用于隐藏对未使用代码的警告                     |
| #![allow(overflowing_literals)]            | 不显示类型转换产生的溢出警告                         |
| 1..10                                      | 表示从1取到10，10不包括                              |
| 1..=10                                     | 表示从1取到10，10包括                                |
| move                                       | 关键字move的作用是将所引用的变量的所有权转移至闭包内 |
| \#[derive(RustcDecodable, RustcEncodable)] | 序列化和反序列化                                     |

