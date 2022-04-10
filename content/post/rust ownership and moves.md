---
title: "阅读Programming Rust第4章"
date: 2022-04-10
tags: [
    "rust",
    "programming rust",
    "ownership",
	"move",
	"copy"
]
comment: true
mathjax: false
toc: true
autoCollapseToc: true
draft: false
---
 
 
 本文是对[Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) 第4章的总结，部分代码与图片来自此书

## 背景
我们通常希望变成语言提供以下特性：

* 我们希望可以控制内存释放的时机
* 我们不希望使用一个对象已经被释放的指针

c/c++与带gc的语言通过不同的机制解决以上问题，但是都各自有不同的缺点
rust使用了`Ownership`与`Borrow`来解决内存管理的问题

##  `Ownership`
* 在rust中一个变量被赋值的时候就拥有了这个值的所有权，一个`Struct`拥有字段的所有权， 一个`Tuple`，`Array`，`Vec`拥有包含元素的所有权。
* 一个值只能被一个变量拥有， 这个变量会对此值的生命周期负责， 当一个变量被`Drop`的时候
它所拥有的所有值也会一起被销毁

以下代码创建了一个`Vec`， 它包含了多个结构体

```rust

 struct Person {

 name: String,

 birth: i32,

 }

 let mut composers = Vec::new();

 composers.push(Person {

 name: "Palestrina".to_string(),

 birth: 1525,

 });

 composers.push(Person {

 name: "Dowland".to_string(),

 birth: 1563,

 });

 composers.push(Person {

 name: "Lully".to_string(),

 birth: 1632,

 });

 for composer in &composers {

 println!("{}, born {}", composer.name, composer.birth);

 }
```
它的内存布局为
![](/owner.png)

在很多情况下，这种一对一的绑定关系显然缺乏灵活性， 在真实的编程当中，对象之间的互相引用，多个对象引用同一个对象是很常见的(比如双向链表， 树结构等等)。所以rust在所有权原则的基础上提供很多其他机制来达成这种灵活性

## `Move`
在rust中，像赋值，对函数传值，函数返回值等操作会使一个变量转移它对值得所有权，在一个变量转移所有权之后， 它会变成未初始化状态， 后续得代码无法访问该变量。比如以下代码
```rust

 let s = vec!["udon".to_string(), "ramen".to_string(), "soba".to_string()];

 let t = s;

 let u = s;

```
在内存中的布局如下所示
![](/move.png)

* `s`在把值转移给t后变为未初始化状态，同时`t`只是对`s`的`值本身`(`Value Proper`)做了赋值。这里的`Value Proper`我理解的意思就是，大部分情况就是一个值的栈上数据， 因为栈上数据不需要做内存管理；当然肯定存在堆上数据做`Move`的情况，总的来说编译器会做一次浅复制
* `Move`操作保证了一个值始终只有一个拥有者，所以有且只有一个人会对这个值的内存管理负责


### 控制流中的`Move`
rust编译器会根据条件语句来判断，一个值是否被`Move`，比如

if/else语句
```rust
let x = String::from("hello, move");
if c {
	f(x);  //这个move允许
}else {
	g(x);  //这个move也允许
}
h(x); //这里无论哪种情况move都已经发生
```

循环语句

```rust
let x = String::from("hello, move");
while c {
	g(x);  //这里编译不会通过，因为x会再第一次move的时候失效
}

```

### 数组中的`Move`
一个`Vec` `V`中的元素属于`V`, 不能将他`Move`给其他变量
```rust

 // Build a vector of the strings "101", "102", ... "105"

 let mut v = Vec::new();

 for i in 101..106 {

 v.push(i.to_string());

 }

 // Pull out random elements from the vector.

 let third = v[2]; // error: Cannot move out of index of Vec let fifth = v[4]; // here too
```

* 当然了一个`Vec`必须可以删除元素， 比如移除队尾数据， 移除数据并且使用队尾填充， 还有著名的`mem::replace`, 这些方法都遵循一个原则就是保证vec在一个连贯的状态,使得`Vec`只在记录`length`的情况下就能维护本身的状态。
由于对外赋值会导致`Move`， 所以一般赋值操作可以使用`Borrow`或者`Clone`
```rust
// Build a vector of the strings "101", "102", ... "105"

 let mut v = Vec::new();

 for i in 101..106 {

 v.push(i.to_string());

 }

 // 1. Pop a value off the end of the vector:

 let fifth = v.pop().expect("vector empty!");

 assert_eq!(fifth, "105");

 // 2. Move a value out of a given index in the vector,

 // and move the last element into its spot:

 let second = v.swap_remove(1);

 assert_eq!(second, "102");

 // 3. Swap in another value for the one we're taking out:

 let third = std::mem::replace(&mut v[2], "substitute".to_string());

 assert_eq!(third, "103");

 // Let's see what's left of our vector.

 assert_eq!(v, vec!["101", "104", "substitute"]);

```

* 值得一提的是 ，一个`Vec`使用`remove`方法删除index处的元素时， 所有右边的元素都会左移，这个操作消耗是很大的。不过这个东西跟rust无关， 所有语言都是这么实现

* 另外在一个for循环中， 我们会将整个`Vec`转移出去， 并且将他打散成各个元素， 后续不管这些元素有没有被再次转移， 整个初始`Vec`已经不可见了
```rust
let v = vec!["a".to_string(), "b".to_string(), "c".to_string()];

  

 for mut s in v {          //move the entire v

 if s == "b".to_string() {

 break;

 }

 }

  

 println!("{}", v[2])  //v is moved, 编译不通过

```


## Copy
在rust中对于简单类型，也使用`Move`语义是没有必要的, 因为简单类型除了自己的`值本身`之外并没有要额外管理的资源，这个资源可能是堆内存，可能是文件句柄等。这个时候对一个变量赋值应该是直接`Copy`就可以了， 事实上他跟`Move`就仅仅是是否清除原变量的`值本身`，原变量是否保持可用

`Copy`类型包含了
* 整数，浮点，字符，布尔，指针等
* `Copy`类型组成的`Tuple`
* `Copy`类型组成的`Array`
* 指定了`Copy`的`Struct`
* 指定了`Copy`的枚举
* 所有复制只需要位拷贝的类型


不包含
*  所有`Drop`的时候需要额外处理的都不是`Copy`类型
* 默认的`Struct
* 默认的`Enum`


## `Rc`和`Arc`
有了`Move`跟`Copy`之后，值的使用仍然比较受限， 很多时候我们希望可以有多个变量持有同一个值，就像其他语言一样。这个时候`Rc`就派上用场了，一个Rc就是在遵循前面几种规则的基础上，创造出的一种引用计数类型。

* 按照`Ownership`的原则， 是不允许多个变量同时持有一个值`T`的。那么怎么做呢？答案就是把值`T`包装到一个容器里， 容器本身遵循`Ownership`, 容器内部来决定何时drop关联的值`T`, 这里这个容器就是Rc.
* `Rc`的共享使用`Clone`来实现, `Clone`操作只是对引用计数+1， 当变量被`Drop`时引用技术-1，当引用技术变为0的时候， Rc所包装的值`T`被释放
* `Rc`所包装的值`T`是不可变的， 这个符合`单个可变`与`多个共享`之间的排斥性,某些情况下可以使用内部可变性容器来解决这个问题， 比如`Cell`
```rust

 let s: Rc<String> = Rc::new("shirataki".to_string());

 let t: Rc<String> = s.clone();

 let u: Rc<String> = s.clone();
```

以下是Rc的内存布局
![](/rc.png)
