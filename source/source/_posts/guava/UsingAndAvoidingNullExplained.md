---
title: Using and avoiding null
date: 2018-10-21 18:53:23
categories: guava中文文档
tags: [guava, 文档]
---

英文原文：[Using and avoiding null](https://github.com/google/guava/wiki/UsingAndAvoidingNullExplained)

> "Null Sucks." -Doug Lea
> 
> "I call it my billion-dollar mistake." -Sir C.A.R.Hoare, on his invention of the null reference

不小心使用`null`会造成各种各样的问题。研究Google的代码库，我们发现95%的集合类不接收`null`值。相比于默默地接收`null`值，快速失败(fail fast)能够更好的帮助开发者。

另外，`null`的含糊不清是令人不快的。返回`null`值很少能明确表示它的意思，例如：`Map.get(key)`返回`null`既可以表示这个`key`在`Map`中值为`null`，又可以表示该`Map`中没有这个`key`。`null`可能表示失败，可能表示成功或者其它情况。相对于`null`使用其他方法能更加让你的意思明确。

但是，有时使用`null`是合适的和正确的。就以内存和速度来说，`null`是廉价的并且在对象数组中是不可避免的。但是在应用程序代码中(注：通常就是我们的业务程序，而不是系统底层库)，`null`是导致代码混乱、困难和各种奇怪Bug的元凶。例如：当`Map.get`返回`null`时，它意味着是值缺失，还是值存在但是该值就是`null`呢！最关键的是，Null本身没有定义它表达的意思。

鉴于这些原因，当有可能使用`null`时，许多Guava的工具类被设计成快速失败(fail fast)，除非工具类本身提供了对`null`友好使用的措施。另外，Guava提供了大量的工具类来帮助你避免使用`null`。

### 具体案例
如果你试图使用`null`作为`set`的值或者作为`Map`的键，劝你不要这样做；如果在查找操作过程中能明确指出某些特殊情况为`null`会更加语义明确。

如果你想使用`null`作为`Map`中的一个值，最好不要这样做；而是维护一个`non-null`键的`set`(该`set`中存储那些值为`null`的Map键)。很容易混淆`Map`中包含一个值为`null`的键和该`Map`中根本就没有这个键这两种情况。所以，很好的方式是将(值为`null`的键和值为`non-null`的键)键分离，并且仔细思考一下当一个值为`null`的键对于你的应用到底意味着什么。

如果你在`list`中使用null，如果这个`list`是稀疏的，你最好使用`Map<Integer, E>`，这样会更加高效，并且能够更加准确地适用你的应用需求。

有时，考虑如果有一个自然的`null object`能被使用。例如：为一个枚举增加特殊的常量来表示null。比如，`java.math.RoundingMode`有一个`UNNECESSARY`值表示“不做任何四舍五入，如果要做四舍五入操作就抛出一个异常”。

如果你真正需要一个null值，但是null值不能和Guava中的集合实现一起工作，你只能选择其他实现。比如，用JDK中的Collections.unmodifiableList替代Guava的ImmutableList。

### Optional
在大部分情况下编程人员使用`null`表示某些缺失：也许已经有一个值、没有值、或者一个值不能被找到。例如，当一个键没有值时`Map.get`会返回`null`。

`Optional<T>`表示可能为`null`的T类型引用。一个`Optional`要么包含一个非null的T引用(在这种情况下我们认为该引用存在)，要么什么都没有(这种情况下我们认为该引用缺失)。所以从不会说该引用为`null`。

``` java
Optional<Integer> possible = Optional.of(5);
possible.isPresent(); //returns true
possible.get(); //returns 5
```

`Optional`无意直接模拟其他编程环境中的”可选” or “可能”语义，但它们的确有相似之处。

我们在下面列出了一些通用`Optional`操作。

### 创建`Optional`
下面是`Optional`都静态方法：

方法|描述
---|---
[`Optional.of(T)`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Optional.html#of-T-)|创建一个含有非null值的`Optional`实例，如果值为null就快速失败(fail fast)。
[`Optional.absent()`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Optional.html#absent--)|返回一个引用缺失的`Optional`实例。
[`Optional.fromNullable(T)`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Optional.html#fromNullable-T-)|创建指定引用的Optional实例，若引用为null则表示缺失。

### 查询方法
`Optional<T>`的非静态方法：

方法|描述
---|---
[`boolean isPresent()`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Optional.html#isPresent--)|如果这个`Optional`是一个非null实例，返回`true`。
[`T get()`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Optional.html#get--)|返回`T`实例，如果该实例缺失，抛出`IllegalStateException`异常。
[`T or(T)`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Optional.html#or-T-)|返回`Optional`中存在的值，如果缺失，返回指定的默认值。
[`T orNull()`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Optional.html#orNull--)|返回Optional所包含的引用，若引用缺失，返回null。
[`Set<T> asSet()`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Optional.html#asSet--)|返回Optional所包含引用的单例不可变集，如果引用存在，返回一个只有单一元素的集合，如果引用缺失，返回一个空集合。

除了这些之外，Optional还提供了几种更方便的实用方法有关详细信息，请咨询Javadoc。

### 使用`Optional<T>`的意义？
除了为`null`赋予一个名字增加可读性以外，最大的优点在于傻瓜式防护。如果想完全编译通过，它强迫你积极的思考什么情况下引用缺失，因此你不得显式地从Optional获取引用。直接使用`null`很容易让人忘掉某些情形，尽管FindBugs可以帮助查找null相关的问题，但是我们还是认为它并不能准确地定位问题根源。

如同输入参数，方法的返回值也可能是null。和其他人一样，你绝对很可能会忘记别人写的方法method(a,b)会返回一个null，就好像当你实现method(a,b)时，也很可能忘记输入参数a可以为null。将方法的返回类型指定为Optional，也可以迫使调用者思考返回的引用缺失的情形。

### 其他处理null的便利方法
当你需要用一个默认值来替换可能的null，请使用[Objects.firstNonNull(T, T)](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/MoreObjects.html#firstNonNull-T-T-) 方法。如果两个值都是`null`，该方法会抛出`NullPointerException`。`Optional`也是一个比较好的替代方案，例如：`Optional.of(first).or(second)`.

还有其它一些方法专门处理null或空字符串：

* [`emptyToNull(String)`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Strings.html#emptyToNull-java.lang.String-)
* [`nullToEmpty(String)`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Strings.html#isNullOrEmpty-java.lang.String-)
* [`isNullOrEmpty(String)`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Strings.html#nullToEmpty-java.lang.String-)

我们想要强调的是，这些方法主要用来与混淆null/空的API进行交互。当每次你写下混淆null/空的代码时，Guava团队都泪流满面。（好的做法是积极地把null和空区分开，以表示不同的含义，在代码中把`null`和空同等对待是一种令人不安的坏味道。

