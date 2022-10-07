---
layout:       post
title:        "Swift如何解决《后台返回了一种让我讨厌的JSON》？"
author:       "beforeold"
header-style: text
catalog:      true
tags:
    - Swift
    - Codable
    - JSON
---

# 背景
浏览到一个关于 Swift Codable  应用的很有意思的案例（见后文参考文档），问题大意如下：
>后端返回的字段与最初协议约定的类型不一致
> - 期望：object 类型
> - 实际： String 类型
> 这导致 Swift Model 声明该字段的不便利

不便利的原因在于：
- 1、如果属性声明为 String 类型，则使用时需要二次加工，从 string 转为期望的 model
- 2、如果声明为 model 类型，因为 Codable 对类型强校验的要求，会直接导致 decode 失败

且不论后端下发 JSON String 的合理性，单从解决问题的思路看，可以尝试如下两种方案。

# 方案 1：自定义 Decodable 实现
Swift Codable 作为 Swift 在 data 与 model 之间编解码的协议层，通过为 model 声明遵循 Codable，编译器自动合成了 model 的 Encodable 和 Decodable 实现，十分便利。在具体的自定义类型中，也有很大的自定义空间。围绕这个案例，进行一个初步探索。
如果返回的数据类型与声明期望的类型不一致时，JSONDecoder 会抛出 error 解码失败，这里可行的方案是为该 model完成自定义的 Decoable 实现。补充类型声明如下：
```Swift
struct Token: Codable {
    let result: Bool?
    
    // 期望为 Body object 类型，返回实际为 string 类型
    let body: Body?
}

struct Body: Codable {
    let serialNumber: String?
    let timestamp: String?
}
```
其中 body 属性为需要自定义实现的字段，按照 Codable 定义，为外层类型 ```Token```，自定义实现如下：
```
/// Token 的自定义 Decoable 实现
init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.result = try container.decodeIfPresent(Bool.self, forKey: .result)
        let jsonString = try container.decodeIfPresent(String.self, forKey: .body)
        let jsonStringData = jsonString?.data(using: .utf8)
        self.body = jsonStringData.flatMap 
            return try? JSONDecoder().decode(Body.self, from: $0)
    }
 }

```
这里主要过程是：
- 自定义 decodable 实现
- 先将这个 key decode 为 string
- 将 string 转 data 后再进行二次 Body 对象的 decode
# 方案 2：进一步封装 JSON String 的解码过程
上述自定义的方案的问题在于无法进行复用，如果有多个字段或者多个 model 存在这个情况，则需要重复拷贝代码。
在实际开发中，更优雅的方案是利用 property wrapper （属性包装器）来完成这一属性的二次加工，达到如下效果，即直接声明经过属性包装的目标类型即可
```Swift
struct Token {
    @JSONString
    var body: Body?
}

```
 其核心在于 property wrapper 的特性：
 - 在 Codable 协议下替代了 body 类型先进行 decoding，
 - 在 wrapper 内部进行了 string -> model 的类型转换
 - 通过 wrappedValue 的编译器支持无缝地获取目标类型的字段
 一个可行的 JSONString property wrapper 实现如下
```
@propertyWrapper
public struct JSONString<Base: Codable>: Codable {
    public var wrappedValue: Base
    public init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        if let string = try? container.decode(String.self),
           let data = string.data(using: .utf8) {
            self.wrappedValue = try JSONDecoder().decode(Base.self, from: data)
            return
        }
        
        self.wrappedValue = try container.decode(Base.self)
    }
    
    public func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        let data = try JSONEncoder().encode(wrappedValue)
        if let string = String(data: data, encoding: .utf8) {
            try container.encode(string)
        }
    }
}
```
相对于前文的自定义 decodable 的实现方案，这个 property wrapper 有额外的支持：
- 支持泛型类型的扩展，从而变得通用 (generic）
- 内部同时兼容了下发 string 或者 object 两种情况
代码将在在 [GitHub](https://github.com/beforeold/BUDCodable/blob/main/Sources/BUDCodable/JSONString.swift) 维护。
# 小结
通过这个例子可以侧面了解到，Codable 的设计结合 Swift 的语言特性有许多可以探索研究的空间，也有不少的关于 Codable 开源扩展的方案，比如 [BetterCodable](https://github.com/marksands/BetterCodable) 等等，这样的可扩展性可以在强类型要求的 API 设计下保有丰富的增强开发空间。

# 参考资料
原案例：[《后台返回了一种让我讨厌的JSON》](https://juejin.cn/post/7022130994241077256)

Apple 开发者文档 [Encoding and Decoding Custom Types](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types)

开源库示例：[BetterCodable](https://github.com/marksands/BetterCodable)