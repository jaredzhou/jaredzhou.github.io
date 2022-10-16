---
title: "Rust References"
date: 2022-04-11
tags: [
    "rust",
    "programming rust",
    "lifetime",
	"reference",
	"borrow"
]
comment: true
mathjax: false
toc: true
autoCollapseToc: true
draft: false
---


 本文是对[Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) 第5章的总结，部分代码与图片来自此书



# 引用类型

rust有三种指针类型

* `引用`
* `裸指针`
* `智能指针`

其中`引用`是一种非拥有指针，即不需要对`被引用对象`的生命周期负责，相反，`引用`的生命周期不能超过`被引用者`的生命周期

很多时候我们并不想转移值的所有权，仅仅需要访问值的某些数据，这个时候就需要使用`引用`了。

引用分为两种类型：

* `共享引用` 可以读取`被引用对象`但是不能修改。同时可以存在多个`共享引用`，`共享引用`本身是`Copy`类型
* `可变引用`可以读取并且修改`被引用对象`，同时只能存在一个`可变引用`， 并且`可变引用`存在的时候， 不能存在任何共享引用，`可变引用`不是`Copy`类型

需要说明的是， `可变引用`存在的时候， 不但是`共享引用不能存在`， `Owner`自己也不能访问和修改`被引用对象`

# 使用引用

以下是使用引用的一些示例

```rust
//共享引用
let x = 10;
let r = &x;
assert!(*r == 10);

//可变引用
let mut y = 32;
let m = &mut y;
*m += 32;
assert!(*m == 64);




```

## 隐式解引用/隐式借引用		

这里需要注意的是由于`解引用`，`借引用`是非常常见的操作，而且编码起来显得非常繁琐，所以rust对`.`操作符的左值做了`隐式解引用`与`隐式借引用`的操作

```rust
//隐式解引用， 接引用
struct Anime {
	name: String,
    bechdel_pass: bool,
};
let aria = Anime {
    name: String::from("Aria: The Animation"),
    bechdel_pass: true,
};

//隐式借引用
aria.print();
(&aria).print();

let anime_ref = &aria;
//隐式解引用
println!("Hello, {}!", anime_ref.first_name);
println!("Hello, {}!", (*anime_ref).first_name);
```

## 嵌套引用

rust中`引用`本身也是一个左值， 所以可以对引用再引用，比如

```rust
struct Point {
	x: i32,
    y: i32,
}
let point = Point { x: 1000, y: 729 };
let r: &Point = &point;
let rr: &&Point = &r;
let rrr: &&&Point = &rr;

//*操作符 循环解引用
let rr0:&&Point = *rrr;
let r0: &Point = *rrr;
//.操作符循环解引用，直到到对应的值
let y: 32 = rrr.y;
```

有个值得注意的点是在[The book](https://doc.rust-lang.org/book/ch15-02-deref.html)中有一段是这么说的

>Note that the `*` operator is replaced with a call to the `deref` method and then a call to the `*` operator just once, each time we use a `*` in our code
>
>

上面的代码表面，对于引用，一个*操作符可以直接`解引用`到多个层级之上的类型。

我认为这是由于当我指定了类型之后， rust编译器会根据目标类型进行`Deref Coercion`, 我姑且叫它`强制类型转换`，`强制类型转`换会无限的将一个引用转成它可以转换到的引用，如果一个类型符合 `T: Deref<Target=U>`


## 引用比较/计算

`引用比较`的是他们背后的值， 引用只能在相同类型直接做比较

`引用运算`可以计算一层引用的背后的值， 多层引用则不能相加

```rust 
let x = 10;
let y = 10;
let rx = &x;
let ry = &y;
let rrx = &rx;
let rry = &ry;
assert!(rrx <= rry);
assert!(rrx == rr
assert!(rx == ry); // their referents are equal 
assert!(!std::ptr::eq(rx, ry)); // but occupy different addresses

//引用相加
let n = rx + ry;  
assert_eq!(n, 20);
let m = rrx + rry; //编译不通过， 这两个变量不能相加

```

## 胖指针

rust有两种常见的`胖指针`

* 对`slice`的引用, 额外包含length信息
* 对`trait object`的引用， 额外包含trait实现地址，即虚表

`胖指针`名字的来源是因为这种指针包含两个字长，一般的指针只包含一个。

指向`unszie type`的指针都是胖指针

#  引用安全

`引用`在rust中是内存安全的， 引用不会产生悬空指针的问题，rust通过`lifetime`来解决这个问题，以下这个段代码展示了违背生命周期的情况

```rust
{
    let r;
    {
        let x = 1;
        r = &x;
    }
    assert_eq!(*r, 1); // bad: reads memory `x` used to occupy
}
```

x在引用r之前被释放了， 这种情况是无法通过编译的, rust对于生命周期的限制如下：

假如我们有一个变量v的引用&v， 使变量r=&v， 我们必须满足

1. 变量v的生命周期必须包含&v的生命周期

2. &v的生命周期必须包含r的生命周期

这两种情况任意一个不满足都会产生悬空指针。

在我看来r是v的一个引用，只需要满足v的生命周期包含r的生命周期即可 

## 引用作为函数入参

一个带`lifetime`的函数是这样子的

```rust
fn f<'a>(p: &'a i32){}
```

这里`f<'a>` 'a是作为函数的参数存在， 我们可以认为是 对于任意生命周期都适用，大部分情况下， 一个实参的周期是大于函数调用的。但是如果函数中存在这种情况，就不行了

```rust
static mut STASH: &i32 = &128;
fn f(p: &i32) {
    unsafe{
        STASH = P;
    }
}
```

STASH的生命周期是全局的， 而根据上面的规则2，p的生命周期必须包含'static。显然对于任意生命周来说这是不满足的。这个时候只有将形参改成'static才能满足这个条件

```rust
fn f(p :&'static i32){
      unsafe{
        STASH = P;
    }
}
```

以上的函数是正确的， 但是适用范围也更小了。

## 引用作为函数返回

以下是一个返回引用的函数

```rust
fn smallest(v: &[i32]) -> &i32 {
   let mut s = &v[0];
   for r in &v[1..] {
       if *r < *s {
           s = r;
       }
   }
   s
}

```

我们忽略了生命周期， 当一个函数只有一个引用作为参数的时候，如果它的返回值包含引用， rust会认为它们有同样的生命周期。现在考虑以下的代码

```rust

let s;
{
    let parabola = [9, 4, 1, 0, 1, 4, 9];
    s = smallest(&parabola);
}
assert_eq!(*s, 0);
```

根据上面的规则1，2， 必须满足parabola的生命周期必须大于s的生命周期， 这段代码中parabola要比s更早drop，所以是不能通过编译的



## 结构体包含引用

结构体必须包含生命周期参数，不然是无法通过编译的

```rust
//含引用结构体
struct S<'a> {
    r: &'a i32
}
//嵌套含引用结构体
struct D<'a> {
    s: S<'a>
}
```



## 省略`生命周期`参数

省略生命周期有以下几种情况：

* 函数不返回引用的情况下， 不需要显示指定`生命周期`
* 如果函数只有一个参数，那么也不需要指定生命周期， 因为rust为认为任何返回引用的`生命周期`都与此参数相同
* 如果函数是一个类型的方法，并且形参是引用类型`self`(&self, &mut self)， rust会认为返回引用的`生命周期`与此相同

## 后续

rust的引用是一个根本又复杂的话题，这里还有很多东西没有讨论清楚， 比如`强制类型转换`， 比如`重借用`等等， 后续还需要进一步的研究









