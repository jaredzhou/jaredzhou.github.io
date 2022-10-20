---
title: "Rust Iterators"
date: 2022-10-20
tags: [
    "rust",
    "programming rust",
     "iterator",
]
comment: true
mathjax: false
toc: true
autoCollapseToc: true
draft: false
---


 本文是对[Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) 第15章的总结，部分代码与图片来自此书

### 一个简单的例子

```rust
//normal style   
let mut sum = 0;
for i in 0..=10 {
        sum += i;
}
println!("{}", sum);
//iterator style
let sum = (0..=10).fold(0, |sum, i| sum+i);
println!("{}", sum);
```

以上两种求和的实现是等价， 基于`iterator`的实现风格对于函数式编程是很常见的， rust也提倡这种方式的写法，不同的是，rust编译器在编译的时候会对这种表达方式进行优化，达到一种`zero cost abstraction`的效果， 比如以上的求和在编译器优化后可能汇编代码使用```11*10/2```这种数学公式来执行。这个意思是不用太担心`iterator`风格的性能问题， rust编译器会进行必要的优化，使得我们使用的时候表达能力与性能兼顾



### `IntoInterator`和`Iterator`

在rust中iterator最基本的两个trait是`IntoInterator`和`Iterator`，他们的定义如下

```rust
trait Iterator {
 type Item;
 fn next(&mut self) -> Option<Self::Item>;
 ... // many default methods
}

trait IntoIterator where Self::IntoIter:Iterator<Item=Self::Item> {
 type Item;
 type IntoIter: Iterator;
 fn into_iter(self) -> Self::IntoIter;
}
```

`IntoIterator`用来生成`Iterator`,  实现`Iterator`可以进行相应的操作， `Iterator`trait包含了很多函数， 但是需要实现的只有一个`next`. 

值得一提的是`IntoIterator`的`into_iter`使用传值的方式，也就表明了这个方法会将原始类型给消费掉， 进行`Copy`或者`Move`, 比如

```rust
let v = vec![1,2,3];
 //into_iter() move the vecotr v
let iter = v.into_iter();
println!("{}", v[0]);
```

以上的代码不能编译通过， 因为v已经被`Move`了， 你可以使用引用来避免这个问题

```rust
let v = vec![1,2,3];
 //以下两种方式等价
let iter = (&v).into_iter();
let iter2 = v.iter();
println!("{}", v[0]);


```

由于这个使用方式很常见， 所以`Vec`直接提供了方法`Vec<T>::iter()`， 他会返回一个生成`&T`类型的`Iterator`

rust中的`for loop`其实是使用`Iterator`来实现的，以下的代码是等价的

```rust
 let v = vec![1,2,3,4];
 for i in &v {
     println!("{}", i);
 }

 //translate to this by compiler
 let mut iter = v.iter();
 while let Some(i) = iter.next() {
     println!("{}", i);
 }
```

很多时候可以使用`IntoIterator`作为泛型的边界来使用，比如

```rust
use std::fmt::Debug;
fn dump<T, U>(t: T)
 where T: IntoIterator<Item=U>,
 U: Debug
{
 for u in t {
 println!("{:?}", u);
 }
}
```

在这个例子里面，表示T必须是一种可以产生`Iterator`， 这个`Iterator<U>`产出的类型`U`必须是实现了`Debug`的



#### `drain`方法

很多`collection`提供了`drain`方法， 它通过`collection`的可变引用返回一个`Iterator`,  当这个`Iteratro` drop之后， 这些被包含的元素就从`colleciton`当中移除了

```rust
 let mut v = vec![1,2,3,4];
 {
     let drain_iter = v.drain(2..4);
 }
 assert_eq!(&v, &vec![1,2]);
```

#### 标准库应用

`Iterator`在标准库中有广泛的应用， 不止在`collection`中， 还在很多其他方面， 比如`io`当中， 使用`Iterator`来处理io流， 尤其在异步io中。

另外对`Iterator`风格代码的使用， 对于非函数式编程习惯的人员来说有一定的门槛，需要多练习才能适应，下面将重点介绍以下`Iterator`适配器



### `Iterator`适配器

rust的`Iterator`适配器非常灵活，下面将逐个介绍

##### `map` and `filter`

这两个应该很好理解

统一处理`Vec`中的元素

```rust
let text = " ponies \n giraffes\niguanas 
    \nsquid".to_string();
let v: Vec<&str> = text.lines()
    .map(str::trim) ////map的FnMut类型为FnMut(I::Item) -> B, Item由上一个Iterator决定， B由这个FnMut决定
    .collect();
assert_eq!(v, ["ponies", "giraffes", "iguanas", "squid"]);
```

过滤`Vec`中的元素

```rust
let text = " ponies \n giraffes\niguanas 
\nsquid".to_string();
let v: Vec<&str> = text.lines()
 .map(str::trim)
 .filter(|s| *s != "iguanas") //FnMut(&Self::Item) -> bool， Item由上一个Iterator决定
 .collect();
assert_eq!(v, ["ponies", "giraffes", "squid"]);

```

需要注意的是

1. rust中adaptor是`lazy`的，也就是说不管chain了多少个adaptor，这些操作只有被消费的时候才执行，具体就是只有adaptor调用了`collect`的时候， 操作才被真正执行
2. adaptor是`zero cost abstraction`， 这意味着rust编译器会将这些chain内联到最后消费的地方， 比如执行的时候就像是一个for循环语句一样，这样做可以明显的提高执行效率

##### `filter_map` and `flat_map`

`filter_map`有点像`map`和`filter`的组合体， 它要么转换一个值，要么丢弃一个值，使用`filter_map`有时候比使用组合方式更加方便和优雅， 比如只有在你转换一个值的时候才能决定是否要丢弃一个值

```rust

let text = "1\nfrond .25 289\n3.1415 estuary\n";
for number in text
 .split_whitespace()
 .filter_map(|w| f64::from_str(w).ok())
{
 println!("{:4.2}", number.sqrt());
}
```

在上个例子中， 你只有做数字转换的时候才知道要不要保留这个值



flat_map就是在每个FnMut中制造出一个`Iterator`, 然后再把这些`Iterator`合并起来， 当然这些都是语义层面的， 真正执行的时候， 编译器会做相应的优化, 可能并不会真的做`Iterator`合并

```rust
    let mut major_cities = HashMap::new();
major_cities.insert("Japan", vec!["Tokyo", "Kyoto"]);
major_cities.insert("The United States", vec!["Portland",
"Nashville"]);
major_cities.insert("Brazil", vec!["São Paulo", "Brasília"]);
major_cities.insert("Kenya", vec!["Nairobi", "Mombasa"]);
major_cities.insert("The Netherlands", vec!["Amsterdam",
"Utrecht"]);
let countries = ["Japan", "Brazil", "Kenya"];
for &city in countries.iter().flat_map(|country|
&major_cities[country]) {
 println!("{}", city);
}
```



##### `flatten`

`flatten`就是值将`Iterator`中的每个元素也当成`Iterator`，然后取第二级中的元素作为输出， flatten通常是值将2层级变为1层级的操作

```rust
let mut parks = BTreeMap::new();
parks.insert("Portland", vec!["Mt. Tabor Park", "Forest Park"]);
parks.insert("Kyoto", vec!["Tadasu-no-Mori Forest", "Maruyama
Koen"]);
parks.insert("Nashville", vec!["Percy Warner Park", "Dragon
Park"]);
// Build a vector of all parks. `values` gives us an iterator producing
// vectors, and then `flatten` produces each vector's elements in turn.
let all_parks: Vec<_> =
parks.values().flatten().cloned().collect();
assert_eq!(all_parks,
 vec!["Tadasu-no-Mori Forest", "Maruyama Koen", "Percy
Warner Park",
 "Dragon Park", "Mt. Tabor Park", "Forest Park"])
```

上面的示例将`Vec`元素打平成`&str`作为结果， 3个`Vec`变成了6个`&str`

`flatten`与`Option`配合使用将会有很妙的效果

```rust
assert_eq!(vec![None, Some("day"), None, Some("one")]
 .into_iter()
 .flatten()
 .collect::<Vec<_>>(),
 vec!["day", "one"]);

```

上面的示例将`Vec<Option>`变成了`Vec<&str>`， 这是因为`Option`本生也实现了`IntoIterator` 返回了非`None`的值，同样的方式也适用于`Result`

有的时候`flatten`与`map`的组合其实就等于`flat_map`， 这种情况下使用`flat_map`会显得更方便与直接

### `其他`

其他还有很多常用的适配器，比如`take`, `skip`， `peekable`, `fuse`, `chain` 等等，暂且不做讨论
