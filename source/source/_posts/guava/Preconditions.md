---
title: Preconditions
date: 2018-10-28 17:21:27
categories: guava中文文档
tags: [guava, 文档]
---
英文原文：[Preconditions](https://github.com/google/guava/wiki/PreconditionsExplained)

Guava提供了大量的方法前置判断的工具类，我们强烈建议以静态的方式引入这些方法。

> Tips: 静态方式引入是指在类中以`import static xx`引入类

每个方法都有三种变种：

1. 无参方法。任何抛出的异常都没有错误信息；
2. 有一个`Object`的额外参数。任何抛出的异常以`object.toString()`的方式表示错误信息；
3. 有一个`String`的额外参数，并且任意个`Object`的参数。这个方法有点像`printf`，但考虑GWT的兼容性和效率，只支持%s指示符。

第三个变种的例子：

``` java
checkArgument(i >= 0, "Argument was %s but expected nonnegative", i);
checkArgument(i < j, "Expected i < j, but %s >= %s", i, j);
```

方法签名(不包括额外的参数)|描述|失败抛出的异常
---|---|---
[checkArgument(boolean)](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Preconditions.html#checkArgument-boolean-)|检测`boolean`是否为`ture`，用于方法的参数检测。|`IllegalArgumentException`
[checkNotNull(T)](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Preconditions.html#checkNotNull-T-)|检测值是否为`null`，该方法直接返回值，所以你可以内嵌使用该方法。|`NullPointerException`
[`checkState(boolean)`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Preconditions.html#checkState-boolean-)|不依赖方法参数而检测对象的状态。例如，一个`Iterator`可能会使用这个方法去检测在调用`remove`方法前是否已经调用过`next`方法。|`IllegalStateException`
[`checkElementIndex(int index, int size)`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Preconditions.html#checkElementIndex-int-int-)|检测`index`是否是list、string或者array的有效索引。一个元素索引有效值从0到`size`(不包括size)。不直接传递list、string或array，仅仅需要`size`值。返回`index`|`IndexOutOfBoundsExeception`
[checkPositionIndex(int index, int size)](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Preconditions.html#checkPositionIndex-int-int-)|与`checkElementIndex(int index, int size)`类似，唯一区别是索引有效值包括`size`。返回`index`|`IndexOutOfBoundsExeception`
[`checkPositionIndexes(int start, int end, int size)`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Preconditions.html#checkPositionIndexes-int-int-int-)|检测`[start, end)`是list、string或array的有效子索引，附带它自己的错误信息|`IndexOutOfBoundsExeception`

相对于其他类型的工具集，例如Apache的Commons，我们更倾向于使用我们自己的前置条件检测工具的原因如下：

* 静态引入方法，Guava方法是清晰、语义明确的。`checkNotNull`清楚的表达了它要做什么和异常怎么抛出
* 在校验成功后`checkNotNull`返回它的参数，可以在构造器中一行完成代码功能：`this.field = checkNotNull(field)`
* 简单，可变参数风格的print异常信息。(这个优点也是我们推荐继续使用`checkNotNull`而不使用[`Objects.requireNonNull`](http://docs.oracle.com/javase/7/docs/api/java/util/Objects.html#requireNonNull(java.lang.Object,java.lang.String))的原因)

在编码时，如果某个值有多重的前置条件，我们建议你把它们放到不同的行，这样有助于在调试时定位。此外，把每个前置条件放到不同的行，也可以帮助你编写清晰和有用的错误消息。