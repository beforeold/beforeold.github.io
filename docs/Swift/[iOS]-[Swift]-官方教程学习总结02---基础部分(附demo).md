## 鸣谢
感谢Apple官方的[Swift教程文档](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/index.html#//apple_ref/doc/uid/TP40014097-CH3-ID0)
此外iBooks也有。

感谢国内小伙伴们翻译的[Swift官方教程中文版](https://github.com/numbbbbb/the-swift-programming-language-in-chinese)。
此外，开源翻译的同学们也在[极客学院](http://wiki.jikexueyuan.com/project/swift/)上同步。


## 推荐体验的方式
体验Swift首选**Xcode**自带的**Playground**，代码编辑完成后会自动编译出结果，十分便利

## 基础部分
历时5小时
初见之后是逐步展开介绍Swift语言的介绍，首先是基础部分，涉及到

- 常量和变量
- 基本数据类型
- 类型安全和类型推断
- 类型别名
- 元组
- 可选类型
- 错误处理
- 断言

## 有意思的知识点

Swift足够简洁，你可以在语句后面是省略分号，除非你要在一行写多个语句（**别这种做！**）

#### 常量和变量
使用`let`声明常量，使用`var`声明变量
常量一旦赋值后就不能再修改，而变量可以，这是常量和变量的本质区别
给常量或者变量赋值第一次赋值的过程就是初始化，声明一个常量/变量时即可进行初始化
初始化时可以不声明变量的类型，因为编译器会根据赋值的值进行自动推断数据类型`type inference`
变量未初始化是无法使用的。
```Swift
let meaningOfLife = 42 // Int
let pi = 3.14159 // Double
let anotherPi = 3 + 0.14159 // Double
```
变量（函数/类等等）的命名可以使用Unicode字符，即中文或者emoji也支持，编程变得更有趣
```Swift
let 你好 = "你好世界"
func 🐶🐶() -> String {
    return 你好
}

🐶🐶()

let 🐱🐱 = "🐱🐱"
print(🐱🐱)

```

#### 数据类型
整数字面量会默认推断为Int类型，小数字面量会默认推断为Double字面量
Double为64位，为Float为32位

Swift中布尔值`Boolean`，用`Bool`表示，只有`true` 和`false`
在if条件的判断时只能传入Bool值，而不能像OC一样传入一个数据做判断
```Swift
let i = 1
// wrong
//if i {
//    
//}

if i == 1 {
    // right
}

```
可以使用下划线来分隔数据来增强可读性
```Swift
let tenThousand = 1_0000; // 一万
```


#### 类型安全
Swift是一门类型安全`type safe`的语言，在操作数据时编译器会进行类型检查`type checking`
如果要同时操作两个不同类型的数据，需要进行类型转换`type casting`
```Swift
let twoThousand:UInt16 = 2000 // 16位非负整数
let one: UInt8 = 1 // 8位非负整数
let towThousandAndOne = twoThousand + UInt16(one)  // 转换后才能使用
```
Swift的注释和OC一样，可以使用 `//`进行一行代码的注释 `/* */`进行多行代码的注释
同时`/* */`支持嵌套，这样大家可以很方便地注释任何代码了
```Swift
/*
/* 注释嵌套 */
*/
```

#### 类型别名
为使数据类型拥有特别的意义，可以为变量起一个别名`typealias`，等同于`C语言`的`typeDef`
```Swift
typealias AudioSample = UInt16

var maxAmplitudeFound = AudioSample.max
```

#### 元组
元组`tuple`是一组数据用小括号表示`(i, j, k)`，元组内的数据可以是任意类型
尤其适合作为有多个返回值的函数的返回值使用，解决返回值意义不明确的问题
```Swift
func returnMultValue() -> (number: Int, description: String) {
    return (1, "1 is good")
}
```
可以结合类型别名typealias来使用元组，这样元组的意义也更明确
```
typealias HTTPResponseTuple = (code: Int, message: String) 
```
元组`tuple`更适合做为临时的数据结构使用，对于更持久和常用、复杂的数据结构可以使用类`class`或者结构体`struct`

#### 可选类型
可选类型optional，可选类型是对基础类型变量的修饰，使用问号`?`在数据类型后进行修饰
可选类型用于处理变量**值缺失**的情况，暗示该变量可能会没有任何值
与OC中最接近的概念是`返回nil`，但是OC中nil只能描述对象，而Swift中可选类型可以修饰所有类型
Swift中的nil不是指针，其表示的是`没有值`
PS:今天实际看代码发现，可选类型更像OC的nullability annotations，也就是nullable这样的变量修饰符
当然也是只是对象才有效，OC中这个特性是为了Swift和OC的混编需要才引入的
参考文章[iOS开发——新特性篇&swift新特性（__nullable和__nonnull）](http://www.cnblogs.com/iCocos/p/4652724.html)

```Swift
var optionalString: String? = nil // 通过在类型后写一个问号
```
获取可选类型的值，需要在变量后使用`!`号，成为强制解析`force unwrapping`，使用可选类型之前为安全考虑最好进行非nil的判断
```Swift
if  optionalString != nil {
    var actualStrin = optionalString!
}
```
可选类型的使用可以结合if判断进行使用，这种用法叫做可选绑定`optional binding`
即先进行可选值的判断是否为空，不为空则赋值后进行if成立条件的语句
```Swift
if let actualNumber = Int(possibleNumber) { // 如果要在if中操作actualNumber let->var
// 注意Int(possibleNumber)构造器函数返回的是一个Int?的可选类型变量
    print("ConvertedNumber \(actualNumber)")
}else {
    print("Unconverted")
}
```
有一种特殊的可选类型，是隐式解析可选类型 `implicitly unwarapper optionals`
在声明时对变量的数据类型使用感叹号`!`进行修饰，该数据类型本质上仍然是可选类型
但使用时不再需要进行强制解析，因为它本身是隐形解析的
不过使用时该变量时仍需要进行安全判断
隐式解析可变类型的使用场景是该变量第一次赋值后声明周期内就不再可能为nil的场景
对于声明周期随时需要进行是否为nil判断的变量，建议使用普通的可变类型变量
```Swift
let assumedSString: String! = "An implictly unwrapped optional String."
let implicitString: String = assumedSString // 不需要感叹号 隐式解析

if let definiteString = assumedSString {
    print(definiteString)
}
```

#### 异常处理和断言
在函数声明中带有`throws`关键字修饰的函数可能会抛出异常
这样的函数的调用需要使用`try`关键字，Swift支持使用`do-catch语句进行错误的判断处理
```Swift
func canThrowAnError() throws {
    // 可能抛出异常
}

do {
    try canThrowAnError() // 需要使用try
}catch {
    // error 的处理
}
```
断言`assertion `如果缺少结果那么出发一个断言来找到值缺失的原因，中止应用,断言允许附加一条调试信息
```Swift
let age = -3
// assert(age > 0, "Age must be bigger than 0")
// assert(age > 0)
```
当代码使用优化编译的时候，断言将会被禁用
例如在Xcode中使用默认的target release配置选项来build时，断言将被禁用
所以放心地使用断言

## 小结
因为有OC的经验，学习Swift还是轻松愉快的
不过结合自己以前学了Python和C#后不用很快忘记的教训，语言学习还是应该要学以致用
多敲多试，并尽早在实际项目中使用起来。

## demo 地址
[demo地址](https://github.com/beforeold/SwiftLearningSummary)
