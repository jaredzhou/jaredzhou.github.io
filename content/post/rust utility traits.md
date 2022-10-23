---
title: "Rust Utility Traits"
date: 2022-10-23
tags: [
    "rust",
    "programming rust",
     "trait",
]
comment: true
mathjax: false
toc: true
autoCollapseToc: true
draft: false
---


 本文是对[Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) 第13章的总结，部分代码与图片来自此书

### Utility Traits

rust中有很多常用的`trait`用来表达一些基础的语言特性， 这些特性贯穿了整个rust语言的设计与应用，熟悉这些`trait`对于编写rust代码是必须的

#### `Drop`

drop发生在一个所有者终结的时候，他所持有的值就要一起被销毁， 这些值包括堆内存，系统资源等等。`drop`通常发生在作用域结束，表达式结束或者`Vec`移除对象的时候。大部分时候rust会自动帮你销毁对象，这正是`Borrow-Check`机制的特别之处，在没有GC的情况下做自动内存管理

`Drop`被定义如下

```rust
trait Drop {
 fn drop(&mut self);
}
```

rust会在值销毁的时候自动调用`drop`， 另外手工调用`drop`是不被允许的， rust编译器会报错

以下例子表明了drop发生的时机

```rust
 struct Appellation {
     name: String,
     nicknames: Vec<String>,
 }

 impl Drop for Appellation {
     fn drop(&mut self) {
         print!("Dropping {}", self.name);
         if !self.nicknames.is_empty() {
             print!(" (AKA {})", self.nicknames.join(", "));
         }
         println!("");
     }
 }

 {
     let mut a = Appellation {
         name: "Zeus".to_string(),
         nicknames: vec![
             "cloud collector".to_string(),
             "king of the gods".to_string(),
         ],
     };
     println!("before assignment");
     //a Owner another value, so the old value drop
     a = Appellation {
         name: "Hera".to_string(),
         nicknames: vec![],
     };
     println!("at end of block");
     //a drop the value it owns
 }
```

在这个例子中`drop`发生了两次， 一次是重新赋值的时候，另外一次是作用域结束的时候

大多数情况，我们并不需要自己实现`Drop`， 但是有些时候rust编译器识别不出值所拥有的资源类型的时候， 你需要手工来释放资源,比如标准库中的文件句柄，通过`drop`来主动释放对于文件句柄的占用

```rust
struct FileDesc {
 fd: c_int,
}

impl Drop for FileDesc {
 fn drop(&mut self) {
 let _ = unsafe { libc::close(self.fd) };
 }
}
```

一个实现了`Drop`的类型，不能同时实现`Copy`,因为`Copy`代表了类型可以通过字节复制来完成拷贝， 这种情况下可能会对于同样的数据`drop`多次， 这种做法是错误的。



#### `Sized`

`Sized`代表了某种类型拥有固定大小， rust中大部分类型都是`Sized`, 数字，字符串，数组等。

`Sized`是一种`Marker Trait`， 这表示它并不需要代码实现，而是由编译器自动实现

以下三种类型是`Unszied Type`

- str
- slice
- trait object(dyn trait)

以下是三种类型的内存示意图

![](/unsized.png)

rust并不能直接把`Unsized Type`放在变量中使用， 通常是通过放在引用或者智能指针后面使用比如`&str`, `&[T]`,`Box::<dyn Writer>`.

除了以上三种类型之外， `struct`也允许在最后一个字段放置一个`Unsized Type`使它自己也变成`Unszie Type`,比如标准库中`Rc`的私有类型`RcBox`

```rust
struct RcBox<T: ?Sized> {
 ref_count: usize,
 value: T,
}

let boxed_lunch: RcBox<String> = RcBox {
 ref_count: 1,
 value: "lunch".to_string()
};
use std::fmt::Display;
let boxed_displayable: &RcBox<dyn Display> = &boxed_lunch;

```

对于一个`Unsize Type`你并不能直接构建， 而是需要通过引用的方式间接实现

另外在泛型边界中， 由于约束`Sized`是很普遍的情况， 所以默认所有泛型`<T>`都会被自动转换成`T:Sized`, 如果你需要`Unsized Type`那么可以显示的指定为`T:?Sized`

#### `Clone`

`Clone`就是对一个值的深复制， 标准库中的大部分类型都是适合复制的，也就是实现了`Clone`，但是某些特定的类型不合适，比如`std::sync::Mutex`是不允许复制的， 因为锁是不能复制的，另外`std::fs::File`也是可以复制的， 但是由于涉及到操作系统调用， 并不能保证复制成功， 所以它也没有实现`Clone`相反它提供了一个`try_clone`方法



#### `Copy`

`Copy`也是一种`Marker Trait`，它表明了一个类型可以进行字节浅复制，在所有`Move`发生的情况下， 如果一个类型是`Copy`类型，那么`Move`不会发生， 取而代之的是复制，也就是原来的值持有者保持可用

`Copy`只可以标记那些能够进行浅复制的类型， 这个约束又编译器来保证， 另外实现了`Drop`的类型不能同时实现`Copy`,两者不能同时存在于一个类型中



#### `Deref` and `DerefMut`

`Deref`是用来表达解引用，解引用与引用是一对相反的操作， 对于引用类型来说可以用`*T`来表示， 为了把这种操作推广到自定义类型，所以有了这一组`Trait` . 但是事实上能用到*的地方其实很少，考虑下面的类型

```rust
struct MyString {
        str: String,
    }

impl Deref for MyString {
    type Target = Stri
    fn deref(&self) -> &Self::Target {
        &self.str
    }

impl DerefMut for MyString {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.str
    }
}
```

这是一个可以解引用为`String`的类型， 也就是说`Deref<MyString>->String`

那么看一下我们可以怎么使用这个特性

```rust
 let mut  my_string = MyString{str: "abc".to_string()};
 //无法通过编译， 因为不能Move &类型,
 let string = *my_string;
 let string = *(my_string.deref());
//通过强制接引用(deref coercion)实现，无需显示写出，也很难写，因为这里面不仅涉及到解引用，还涉及到借引用， 反正是神奇的dot operator的黑魔法
  _ = my_string.len();
//必须实现DerefMut才可以
 *my_string = "def".to_string();
 *(my_string.deref_mut()) = "def".to_string();
//跟下面的是一个意思
let mutstr = &mut "123".to_string();
*mutstr = "def".to_string();
```

从上面的例子可以看出，`Deref`只是参与了*操作， 两者并不等价， *v操作可能等于`*(v.deref())`也可能等于`v.deref().copy()`, 具体怎么`desugar`还要看使用情况。我们需要在直觉方面保持一致

需要指出的是`Box<T>`是一个比较特别的类型， 它可以通过`*Box<T>`将T给`Move`出来

```rust
let mut a = Box::new("abc".to_string());
//可以编译通过， 应该是编译器特性
let b = *a;
```

[详细情况可以参考这个RFC](https://github.com/rust-lang/rfcs/issues/997)

#### `AsRef`和`AsMut`

这两个trait允许从实现了`T:AsRef<U>`的类型T中获取&U, 也是一种满足灵活性的设计， 比如使用`AsRef<U>`作为函数参数， 就允许传入更多的类型
需要说明的是， 泛型并不会做强制类型转换， 比如

```rust
impl AsRef<[u8]> for str {}
```

str类型实现了AsRef, 但是通常情况下， 我们是使用&str来进行编码， 这种情况编译器将在类型检查的时候无法通过， 为了解决这个问题， 标准库包含了一个通用实现

```rust
impl<'a, T, U> AsRef<U> for &'a T
 where T: AsRef<U>,
 T: ?Sized, U: ?Sized
{
 fn as_ref(&self) -> &U {
 (*self).as_ref()
 }
}
```
这使得， 任何`T: AsRef<U>`都同时有 `&T:AsRef<U>`, 也就是通过T，&T都可以得到&U类型



#### `Borrow`和`BorrowMut`

`Borrow`与`AsRef`类似，也是获取`T: AsBorrow<U>`,也是通过T获取&U的方式.所不同的是， AsBorrow通常约定T与U之间有某种联系,比如`HashMap<K,V>`

```rust
use std::borrow::Borrow;
use std::hash::Hash;

pub struct HashMap<K, V> {
    // fields omitted
}

impl<K, V> HashMap<K, V> {
    pub fn insert(&self, key: K, value: V) -> Option<V>
    where K: Hash + Eq
    {
        // ...
    }

    pub fn get<Q>(&self, k: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq + ?Sized
    {
        // ...
    }
}
```

`HashMap<K, V>`中存储的是K,V类型， 但是它允许使用Q满足`K: Borrow<Q>`来查询K对应的值， 这必然需要提供一下约束来保证实现， 因为HashMap与hash, eq,ord等运算符有关系， 比如要保证hash(K1)==hash(K2)的时候hash(Q1)==hash(Q2).
当你能满足K与Q的Eq,Ord, Hash都一致的时候就实现`AsBorrow`,否则就实现`AsRef` .另外Borrow也有一些通用实现，保证了K, &K, &mut K 作为K的时候， &K作为Q，也就是实现了`K: AsBorrow<K>`, `&K: AsBorrow<K>`, `&mut K: AsBorrow<K>`，当然还有更多的智能指针比如`Box<K>: AsBorrow<K>`

#### `From`和`Into`
这是一组一样用途的Trait，实现了`From`就会自动实现`Into` . `T: From<U>`表示可以从U转换成T类型。`From`的一个典型用途就是`?`操作符， 它可以将错误进行转换， 符合函数需要的错误类型，从而简化代码
```rust
//所有实现了std::err::Error的类型都可以转换成Box<dyn Error>
impl<'a, E: Error + Send + Sync + 'a> From<E>
 for Box<dyn Error + Send + Sync + 'a> {
 fn from(err: E) -> Box<dyn Error + Send + Sync + 'a> {
 Box::new(err)
 }
}

type GenericError = Box<dyn std::error::Error + Send + Sync +
'static>;
type GenericResult<T> = Result<T, GenericError>;
fn parse_i32_bytes(b: &[u8]) -> GenericResult<i32> {
 Ok(std::str::from_utf8(b)?.parse::<i32>()?)
}
```

#### `ToOwned`

`ToOwned`的定义如下

```rust
trait ToOwned {
 type Owned: Borrow<Self>;
 fn to_owned(&self) -> Self::Owned;
}

```
它允许从&T中生成一个`U: Borrow<T>`, 这使得它的使用范围比`Clone`要大， 你可以从多种可能的类型中选择一种， `Clone`只能返回`T`类型，比如`[T]`可以实现成`ToOwned<Owned=Vec<T>>`, 因为`Vec<T>:Borrow<[T]>`， 相反[T]因为是`Unsized Type`, 是不允许实现`Clone`的

#### `Cow`
`Cow`是`Copy On Write`的意思，它的定义是一个`Enum`类型
```rust
enum Cow<'a, B: ?Sized>
 where B: ToOwned
{
 Borrowed(&'a B),
 Owned(<B as ToOwned>::Owned),
}
```
可以看出来，它利用了`Borrow`和`ToOwned`两种Trait来实现， 一个Cow即可以是一个&B， 也是可以是`T:  Borrow<B>`, 其中T是值， &B是引用， 当必要的时候这两个之前可以互相转换

```rust
use std::path::PathBuf;
use std::borrow::Cow;
fn describe(error: &Error) -> Cow<'static, str> {
 match *error {
 Error::OutOfMemory => "out of memory".into(),
 Error::StackOverflow => "stack overflow".into(),
 Error::MachineOnFire => "machine on fire".into(),
 Error::Unfathomable => "machine bewildered".into(),
 Error::FileNotFound(ref path) => {
 format!("file not found: {}", path.display()).into()
 }
 }
}

```

