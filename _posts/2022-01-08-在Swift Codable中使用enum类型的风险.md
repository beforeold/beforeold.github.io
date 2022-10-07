---
layout:       post
title:        "在Swift Codable中使用enum类型的风险"
author:       "beforeold"
header-style: text
catalog:      true
tags:
    - Swift
    - Codable
    - Enum
---

# 背景
枚举（enum）类型因其良好地表达对象不同的情况（case），是开发中的常用类型。
例如：

```Swift
enum OrderState: Int {
    case new = 1
    case payed = 2
    case done = 3

}
```

对于原始值为 Int 类型的 enum 类型，编译器可为其自动合成 Codable 实现。

```Swift
extension OrderState: Codable { }

struct Order: Codable {
    let id: Int
    let state: OrderState
}
```

不难猜到，一个 enum 类型的 codable 的模拟实现过程：
- 将是将 value 解码为 Int
- 尝试将 Int 构造为 enum 类型

```Swift
extension OrderState: Codable {
    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let int = try container.decode(Int.self)
        self = OrderState(rawValue: int) ?? _______
    }
}
```

然则在 Codable 中使用 enum 类型时有一处潜在的风险。原因是从客户端当前的角度看例子中 OrderState 是固定的，然而后端的 API 在将来则可能迭代拓展，如果下发的数值超过了当前声明的范围，例如```case cancel = 4```，则导致 decoder 失败。
这一点就对应到了上述模拟实现过程中的从 Int 到 OrderState 的构造可能失败。

# 解决方案：使用 struct 替代
enum 的原始值支持 codable 的原因是其遵循了 RawReresentable 协议，在 Codable 中提供了默认实现：

```Swift
extension RawRepresentable where Self : Decodable, Self.RawValue == Int {

    /// Creates a new instance by decoding from the given decoder, when the
    /// type's `RawValue` is `Int`.
    ///
    /// This initializer throws an error if reading from the decoder fails, or
    /// if the data read is corrupted or otherwise invalid.
    ///
    /// - Parameter decoder: The decoder to read data from.
    public init(from decoder: Decoder) throws
}
```

借助这样的实现，可以用使用 struct 遵循 RawReprentable，可以获得：
- 可以规避 case 失败的问题，支持任意 int 下发
- 通过补充静态属性代替具体的 case 声明
例如：

```Swift
struct OrderStateStruct: RawRepresentable, Codable {
    let rawValue: Int
    
    static let new = OrderStateStruct(rawValue: 1)
    static let payed = OrderStateStruct(rawValue: 2)
    static let done = OrderStateStruct(rawValue: 3)
}
```

# 小结
Swift 的强类型特性设计在 Codable 中突出的体现，在设计参数类型时需要格外注意。有几点注意：
- 类似的方案，也可用于原始值为 string 类型的枚举场景
- 使用 struct 的方式，在 swift 可以继续保留 swift case 的支持，不同的是会有 default case 的存在，因为不是可穷举的，这在未来新增 case 时，需要在用到该枚举的地方逐个查找并补充，相对 enum 类型没有编译器的支持，权衡得失，问题不会太大。 
- 如果坚持使用 enum，那么就需要考虑到 decode 失败的兜底处理，可以从自定义实现 Decodable 等角度去思考和拓展，例如评论中有同学提到的，将不符合 1、2、3 的 case 统一设置为 case unkown = 4 进行处理。

```Swfit
/// 新增兜底 case
case unkown = 4

/// 自定义实现 decodable 兜底
extension OrderState: Decoable {
    init(from decoder: Decoder) {
        let container = try decoder.singleValueContainer()
        let int = try container.decode(Int.self)
        self = OrderState(rawValue: int) ?? .unkown
    }
}
```


# 参考资料
Sundel：[Codable synthesis for Swift enums](https://www.swiftbysundell.com/articles/codable-synthesis-for-swift-enums/)

Onevcat：[使用 Property Wrapper 为 Codable 解码设定默认值](https://onevcat.com/2020/11/codable-default/)