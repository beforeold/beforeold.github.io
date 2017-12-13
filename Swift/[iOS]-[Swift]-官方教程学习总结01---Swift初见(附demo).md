## 鸣谢
感谢Apple官方的[Swift教程文档](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/index.html#//apple_ref/doc/uid/TP40014097-CH3-ID0)
此外iBooks也有。

感谢国内小伙伴们翻译的[Swift官方教程中文版](https://github.com/numbbbbb/the-swift-programming-language-in-chinese)。
此外，开源翻译的同学们也在[极客学院](http://wiki.jikexueyuan.com/project/swift/)上同步。


## 推荐体验的方式
体验Swift首选**Xcode**自带的**Playground**，代码编辑完成后会自动编译出结果，十分便利


## Swift初见
历时7个小时，将`Swift初见`看并敲了一遍，通过这个初见的章节对Swift这门语言有了直观的认识，有以下感受

- 力争简洁，从字符串拼接，log的方式，变量的声明和类型推断功能，各种符号或语句的省略等等能看出来
- 同时支持了面向过程和对象编程也支持创建类
- 具有高扩展性，几乎每种类型都有函数的支持，基础类型、枚举，结构体、类等都能实现扩展`extension`，协议`protocol`和自定义函数`func`，大大方便了函数式编程，

初见主要简单介绍下面几个内容

- 简单值（Simple Values）
- 控制流（Control Flow）
- 函数和闭包（Functions and Closures）
- 对象和类（Objects and Classes）
- 枚举和结构体（Enumerations and Structures）
- 协议和扩展（Protocols and Extensions）
- 错误处理（Error Handling）
- 泛型（Generics）

## 有意思的知识点
`let`用于声明常量 `var`用于声明变量，如果不显示声明类型的话，会根据第一次赋值时自动推定类型，因此省掉了在声明变量时先声明类型了，代码里省掉了类型声明
```Swift
let a = 3 // 常量的声明，等同于
let a2: Int = 3

var b = 3.0 // 变量的声明，等同于
var b2: Double = 3.0

```

字符串、数组、字典都没有了OC使用的@符号，而数组和字典都用[]符号
```Swift
"Hello world!"  // 字符串 
[1, 2]    ["A", "B"]  //数组 
 ["key":"value"] //字典
```



很方便地log出变量，字符串也可以很方便地拼接
```Swift
let greetingToFloat = "Hello \(float)" + "Tom"

let label = "The width is"
let width = 94
let widthLabel = label + String(width)

let apples = 3
let oranges = 5
print("apples number: \(apples)")
let appleSum = "I have \(apples) apples"
let fruitSum = "I have \(apples + oranges) fruits"

```

if-else条件判断，条件的小括号可以省略，而满足条件的代码块大括号不能省略
使用`if-let`句式进行条件判断还可以结合变量赋值一起使用，这里不展开了  
```Swift
if true {
    // do something
}
```

Swift版的三目运算符，仍旧使用`condition ? a : b `用法，但没有了`?:`连用的做法，改用`??`，同时可以结合`optional`类型使用，`optional`类型不太理解，不展开了
```Swift
let a = true ? "Good" : "Bad"
let b = "Good" ?? "Bad"

var string: String? = nil // optional类型
let c = string ?? "placeholderString"
```
switch的判断也不再局限于整数，可以是任意类型，条件也不再是相等，可以是自定义的条件，同时也可以结合`let`赋值使用，每个`case`都是独立的，不需要使用`break`关键字
```Swift
let vegetable = "red pepper"
switch vegetable {
case "celery":
    print("Add some rainsins and make ants on a log.")
    
case "cucumber", "watercress":
    print("That would make a good tea sandwich")
    
case let x where x.hasSuffix("pepper") :
    print("is it a spicy \(x)");
    
default:
    print("Everything tastes good in soup.")
}
```

集合内数据的类型可以是任意类型，但需要保持一致，比如数组`[Int]`只能保存Int类型的数据
遍历数组统一使用`for-in`，其中`0 ..< i`表示，下限是0， 上限是`i-1`，而`0 ... i`表示下限是0，上限是`i`。
```Swift
let list = [1, 2, 3]
for i in list {
      print(i)
}

// 等同于
for j in 0 ..< list.count {
    let num = list[j]
    print(num)
}

// 字典的遍历
let interestingNumbers = ["Prime":[2,3,5,7,11,13],"Fibonacci":[1,1,2,3,5,8],
"Square":[1,4,9,16,25,36]]

var largest = 0
var largestKind = ""
for(kind, numbers) in interestingNumbers {
    for number in numbers {
        if number > largest {
            largest = number
            largestKind = kind
        }
    }
}
print("largest~> \(largest) kind~> \(largestKind)")




```
Swift也支持闭包closure，就像OC中的block，而官方的一个说法是说函数是一种特殊的闭包，值得思考


关于类型、枚举、结构体，以及协议、拓展的使用具体实例场景去体验
Swift支持try-catch-throw错误处理

`defer`关键字修饰的代码块会在函数体内结束后一定会执行的代码，无论是否抛出异常，而`defer`代码在函数体内的位置并不影响其调用的顺序，比如
```Swift
var fridgeIsOpen = false
let fridgeContent = ["milk", "eggs", "leftovers"]

func fridgeContains(itemName: String) -> Bool {
    fridgeIsOpen = true
    defer {
        fridgeIsOpen = false  // 一定会关闭冰箱
    }
    
    let result = fridgeContent.contains(itemName)
    return result
}

fridgeContains("Bananer")
print(fridgeIsOpen)

```

## 难点
下面几个特性是需要在进一步的学习和开发中去学习的难点
- 条件判断的let句式，是常用的一种方式，可以在条件语句中获取值进行处理
- 闭包的使用
- 泛型的使用



## 有意思的Swift
```Swift
class __ {
    var __: Int = 0;
}

extension Int {
    var __: Int {
        return self
    }
}

let ___ = 3

var i = __().__.__  
// 1、创建类__的一个实例
// 2、获取其__属性
// 3、调用该属性的扩展属性
print(i)
```

## demo地址
[demo](https://github.com/beforeold/SwiftLearningSummary)
