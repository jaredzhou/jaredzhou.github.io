---
title: "阅读Programming Rust第5章(续)"
date: 2022-10-15
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



# 共享引用与可变引用

虽然可以同时存在多个共享引用，但是当共享引用存在的时候，不但不能够存在可变引用，甚至你也不能修改或者`move` 拥有者本身，比如

```rust
let v = vec![4, 8, 19, 27, 34, 10];
let r = &v;
let aside = v; // move vector to aside
r[0]; // bad: uses `v`, which is now uninitialized
```

上面的`v` 在被引用的时候`move`给了`r`， 这种操作是编译器禁止的

一个场景是在编程当中经常出现的，比如你需要复制字符串自身,使用rust，你可能会这样实现

```rust
fn extend(vec: &mut Vec<f64>, slice: &[f64]) {
 for elt in slice {
 vec.push(*elt);
 }
}

let wave = vec![0.0, 1.0];
extend(&mut wave, &wave);

```

但是这种操作是被编译器禁止的，由于修改`wave`的时候很可能改变它的内存地址，比如扩充大小，这个时候`slice`指向 了已经被摧毁的内存地址，程序就会出现访问无效内存的问题，以下是示意图

![](/sharedmut.png)

这些编译错误使我们重新看一下rust的引用规则

- 共享访问

  被共享引用的值是只读的，在共享引用的周期内， 这个值不能被改变，即使是通过变量本身也不行

- 排他访问

​				可变引用是排他的， 在可变引用周期内， 这个值甚至不能被访问，唯一的例外就是可以`reborrow`这个可变引用

一下是一些体现这些规则的例子

### 共享引用期间不能修改

```rust
    let mut y = 10;
    let r = &y;
    y = 11;  //can not change y
    println!("{}", r);
```

### 可变引用期间不能问

```rust
   //can not use of mut borrowed y, even it is copy type
      let mut y = 10;
      let r = &mut y;
      
      let z = y;  //can not access y
      print!("{}, {}", r, z);
```

### 从共享引用`reborrow`

```rust
 let mut w = (107, 109);
    let r = &w;
    let r0 = &w.0;

    println!("{:?}, {}", r, r0);
```

### 从可变引用`reborrow`

```rust

  let mut w = (100, 101);
    let m = &mut w;
    let m0 = &mut m.0; //reborrow mutable from m
    // let m0 = &mut w.0; //not possible, can only reborrow from m
    let r1 = &m.1;  //reborrow shared from m

    // let v = w.0; //can not accessed, because of mut borrowed by m
    print!("{:?}, {}",r1, m0);
```

这些限制可能看起来很奇怪，也会限制灵活性。 但是能排除一些安全隐患， 最典型的就是同时操作自身的例子，比如操作一个文件，我们很可能会复制自身

```rust
struct File {
 descriptor: i32
}
fn new_file(d: i32) -> File {
 File { descriptor: d }
}
fn clone_from(this: &mut File, rhs: &File) {
 close(this.descriptor);
 this.descriptor = dup(rhs.descriptor);
}


let mut f = new_file(open("foo.txt", ...));
...
clone_from(&mut f, &f);
```

如果以上代码能够执行，那么显然跟我们预期不太一样， 一个我们希望复制的文件已经被关闭了

这些现在也会在并发编程的时候带来一些好处， 因为并发中的竞争就是因为多个线程对于同个变量的修改导致的。但是rust中并不允许同时存在多个可变引用。



# 对象海

垃圾回收机制的引入通常导致了对象海的产生，开发者会在代码中将各个对象互相引用， 这样做可能会提高一点开发效率，但是由于缺乏良好的设计，随着代码量的增加，系统很快就会变成一团乱麻，难以维护， rust严格的所有权机制会使开发者提前思考系统设计，写出更好，更加可维护的程序
