title: Swift中使用reduce函数的一个小示例
tags:
  - Swift
  - reduce
categories:
  - Swift
author: Steven Hu
date: 2018-03-16 22:12:00
---

今天看到一篇通过一个比较有意思的示例来讲解 reduce 函数使用的文章 [reduce all the things](http://appventure.me/2015/11/30/reduce-all-the-things/)，自己也依葫芦画瓢的在 Swift 4.0 下实现了一遍 :)

示例大致是这样的: 

如下为美国各个城市的 persons 列表，其中每个 person 的结构包括姓名、所属城市（其中城市名字符串的逗号之后为州名，如 CA 为加利福尼亚州）、平均年龄。请使用 map/flatmap/filter/reduce 等函数来封装一个查询指定州的总人数和平均年龄，其中函数输入参数为州名，persons 列表。

```swift
let persons: [[String: Any]] = [
    ["name": "Carl Saxon", "city": "New York, NY", "age": 44],
    ["name": "Travis Downing", "city": "El Segundo, CA", "age": 34],
    ["name": "Liz Parker", "city": "San Francisco, CA", "age": 32],
    ["name": "John Newden", "city": "New Jersey, NY", "age": 21],
    ["name": "Hector Simons", "city": "San Diego, CA", "age": 37],
    ["name": "Hector Simons", "city": "Douglas County, CO", "age": 39],
    ["name": "Brian Neo", "age": 27],
]
```

这个示例如果是用 map/flatmap/filter 等函数组合起来也是可以实现的，只是比较啰嗦不够优雅，而且在这个 **persons** 量级上升后，性能也会不那么乐观（主要涉及到对整个 **persons** 的遍历次数，使用 **reduce** 只需要对 **persons** 遍历一次，具体可以参考那篇文章）。

<!--more-->

先来简单看下 reduce 在 Swift 中的定义：

```swift
    /// Returns the result of combining the elements of the sequence using the
    /// given closure.
    ///
    /// Use the `reduce(_:_:)` method to produce a single value from the elements
    /// of an entire sequence. For example, you can use this method on an array
    /// of numbers to find their sum or product.
    ///
    /// The `nextPartialResult` closure is called sequentially with an
    /// accumulating value initialized to `initialResult` and each element of
    /// the sequence. This example shows how to find the sum of an array of
    /// numbers.
    ///
    /// ... ...
    ///
    /// - Parameters:
    ///   - initialResult: The value to use as the initial accumulating value.
    ///     `initialResult` is passed to `nextPartialResult` the first time the
    ///     closure is executed.
    ///   - nextPartialResult: A closure that combines an accumulating value and
    ///     an element of the sequence into a new accumulating value, to be used
    ///     in the next call of the `nextPartialResult` closure or returned to
    ///     the caller.
    /// - Returns: The final accumulated value. If the sequence has no elements,
    ///   the result is `initialResult`.
    public func reduce<Result>(_ initialResult: Result, _ nextPartialResult: (Result, Element) throws -> Result) rethrows -> Result
```

在注释中很清晰的描述了 reduce 的算法逻辑，给定一个初始化的 result，然后不断的通过调用 nextPartialResult 闭包来计算下一个 result，该闭包的参数为 (上一次执行后的 result 和 collection 中的元素)，然后返回的 Result 再继续调用 nextPartialResult 闭包，直到返回最终的 result 值。

下面是在 swift 4.0 上的实现代码：

```swift
/// 从 persons 人口列表中查询指定州(state)的(人口总数和平均年龄)
///
/// - state: 州缩写，如：CA
/// - persons: 需要查询的总人口列表
func query(state: String, persons: [[String: Any]]) -> (count: Int, age: Float) {
    
    // 使用类型别名，让代码看上去更简洁易懂些
    typealias Acc = (count: Int, age: Float)
    
    // 将 reduce 后的结果（为一个 tuple）存储为临时变量
    // 结果中的 count 为指定州的总人口，age 为总的年龄书
    let u = persons.reduce((count: 0, age: 0.0)) { (acc: Acc, person) -> Acc in
        guard let personState = (person["city"] as? String)?.components(separatedBy: ", ").last,
            let personAge = person["age"] as? Int,
            personState == state
            
            // 如果没有城市(city)，也没有年龄(age)，
            // 或者非指定的州，则直接返回当前的 acc
            else { return acc}
        return (count: acc.count + 1, age: acc.age + Float(personAge))
    }
    
    // 最终再计算出平均年龄
    return (count: u.count, age: u.age / Float(u.count))
}
```

这里使用 tuple 作为 accumulator 来将关联数据与 reduce 关联起来。

调用时只需要传入需要查询的州缩写（如：CA）即可查询该州的人口总数和平均年龄：

```swift
query(state: "CA", persons: persons)  // (count: 3, age: 37.0)
query(state: "NY", persons: persons)  // (count: 2, age: 36.0)
```
