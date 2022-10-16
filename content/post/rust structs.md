---
title: "Rust Struct"
date: 2022-10-16
tags: [
    "rust",
    "programming rust",
	"struct"
]
comment: true
mathjax: false
toc: true
autoCollapseToc: true
draft: false
---


 本文是对[Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) 第9章的总结，部分代码与图片来自此书



# Struct

### 基本用法

在rust中， struct分为三种类型，分别为`named`, `tuple-like`, `unit-like`

```rust
struct User {
    id: usize,
    name: String,
    manager: Option<Rc<User>>,
}

struct Point(usize, usize);

struct empty;
```

关联方法可以使用`self`, `&self`, `&mut self`,  `Box<Self>`,  `Rc<Self`>等self参数

```rust
impl User {
    fn new(id: usize, name: String) -> Self {
        User { id: id, name: name, manager: None }
    }

    fn get_name(&self) -> String {
        self.name.clone()
    }

    fn set_name(&mut self, name: String) {
        self.name = name;
    }

    fn move_user(self) -> User {
        self
    }

    fn manage(self: Rc<Self>, user: &mut User) {
        user.manager = Some(self);
    }
}
```

可以这样调用这些方法

```rust
fn main() {
    let mut member = User::new(1, "jared".to_string());
    let manager = User::new(2, "jerry".to_string());
    //&mut self
    member.set_name("abc".to_string());
    //&self
    member.get_name();
    //Rc<Self>
    Rc::new(manager).manage( &mut member);
    //Box<Self>
    let m2 = Box::new(member);
    m2.get_name();
}
```

调用函数时， 类型会自动转换(`implicit conversion`), 当然是在满足规则的情况下， 一个`&self`是没办法自动转换成一个`&mut self`， 相反则是成立的



### 内部可变性

有的时候， 你需要在程序中共享一些struct， 比如说配置参数，由于使用Rc， 这个时候这个配置结构是不可变的。但是很多时候， 你还是希望可以修改它的某些属性， 这个时候就要用到内部可变性

```rust
//配置参数
struct Config {
    path: RefCell<String>,
    timeout: 5,
}

//实例
struct Instance {
    conf: Rc<RefCell<Config>>
}

//共享配置参数
 let config = Rc::new(Config{path: RefCell::new("/abc".to_string()),timeout: 5});
 let mut ins = Instance{conf: config.clone()};
 let ins2 = Instance{conf: config};

//Config本身是immutable的， 但是可以通过内部可变性，改变path的值
 let _ = ins.conf.path.borrow_mut().replace("abc", "xyz") ;
 println!("{}", ins2.conf.path.borrow());
```

内部可变性的问题在于， 它打破了rust编译期的借用规则， 但是在运行期，如果它同时存在一个可变引用与不可变引用，程序会`panic`, 也就是说在运行期它依然要遵循借用规则
