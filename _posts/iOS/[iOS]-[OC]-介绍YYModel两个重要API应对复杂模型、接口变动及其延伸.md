## 背景
数据转模型，是当前 iOS 开发的流行实践。对比数据交付到业务层的方式，```Foundation```框架下的字典、数组等 ```JSONObject```以字符串硬编码（也可以通过自定义常量字符串避免硬编码调用），不如模型对象交付来得直观和值传递的便利。其中，[YYModel](https://github.com/ibireme/YYModel) 是常用的现下一种数据转模型方案。

在实际的开发中，因为服务端业务变动，两个接口返回的数据内容基本一致，但有个别字段的命名不一致，比如一个接口返回是 ```from_beg_time```，一个返回是 ```from_time_beg```，为了这个特例去专门新增一个新的类显然很不值得，初期的方案是为这个类声明两个不同属性，只取有值的内容，如下：
```Objective-C
NSNumber *timestamp = model.from_beg_time ?: model.from_time_beg;
```
后来，为了避免疏漏，为此再声明一个专门的```getter```属性用于取值，并弃用旧的属性避免调用:
```
@property (nonatomic, strong) NSNumber *from_time_beg_get;
@property (nonatomic, strong) NSNumber *from_time_beg __deprecated_msg("");
@property (nonatomic, strong) NSNumber *from_beg_time __deprecated_msg("");

@implementation SomeModel
- (NSNumber *)from_time_beg_get {
    return self.from_time_beg ?: self.from_beg_time;
}
@end

```
再后来进一步研究 YYModel 的 API 后有更新的发现，可提高这些业务场景下的应对能力。

## 介绍两个重要API及其使用

> 以下 API 及其注释均来自 [YYModel](https://github.com/ibireme/YYModel) 源码

#### API-1 自定义容器属性内的对象类型
```Objective-C

/**
 The generic class mapper for container properties.
 
 @discussion If the property is a container object, such as NSArray/NSSet/NSDictionary,
 implements this method and returns a property->class mapper, tells which kind of 
 object will be add to the array/set/dictionary.
 
  Example:
        @class YYShadow, YYBorder, YYAttachment;
 
        @interface YYAttributes
        @property NSString *name;
        @property NSArray *shadows;
        @property NSSet *borders;
        @property NSDictionary *attachments;
        @end
 
        @implementation YYAttributes
        + (NSDictionary *)modelContainerPropertyGenericClass {
            return @{@"shadows" : [YYShadow class],
                     @"borders" : YYBorder.class,
                     @"attachments" : @"YYAttachment" };
        }
        @end
 
 @return A class mapper.
 */
+ (nullable NSDictionary<NSString *, id> *)modelContainerPropertyGenericClass;
```
如果模型的一个属性是容器类型，比如数组、集合或者字典，则这个模型类可通过实现此```类方法```来确定容器内对象的类，通过映射关系告知 YYModel 实现容器内的数据转化为模型，值得注意的三点：
- 返回一个字典对象，key 对应 value 可以返回对应的类对象```[SomeClass class] 等同 SomeClass.class```或者类名字符串```@"SomeClass"```
- 建议在属性声明时利用泛型便于业务方快速识别容器内对象的类型（题外话，如若 OC 语言本身支持泛型类型获取就可省去这一环节了），如：
```Objective-C
@property (nonatomic, copy) NSArray <SomeClass *> *objects;
```
- 对于属性为其他模型类的情况，直接对应声明为对应模型类即可实现类型的映射，不需要为此做其他映射定制，如：
```Objective-C
@propety (nonatomic, strong) OtherClass *info;
```


#### API-2 自定义属性名称对应到数据中的 key，乃至keyPath以及多个key。
```Objective-C
/**
 Custom property mapper.
 
 @discussion If the key in JSON/Dictionary does not match to the model's property name,
 implements this method and returns the additional mapper.
 
 Example:
    
    json: 
        {
            "n":"Harry Pottery",
            "p": 256,
            "ext" : {
                "desc" : "A book written by J.K.Rowling."
            },
            "ID" : 100010
        }
 
    model:
        @interface YYBook : NSObject
        @property NSString *name;
        @property NSInteger page;
        @property NSString *desc;
        @property NSString *bookID;
        @end
        
        @implementation YYBook
        + (NSDictionary *)modelCustomPropertyMapper {
            return @{@"name"  : @"n",
                     @"page"  : @"p",
                     @"desc"  : @"ext.desc",
                     @"bookID": @[@"id", @"ID", @"book_id"]};
        }
        @end
 
 @return A custom mapper for properties.
 */
+ (nullable NSDictionary<NSString *, id> *)modelCustomPropertyMapper;
```
一般而言，模型的属性与 JSON 中的 key 是一一对应的，但总有例外的情况，比如常见的```id```、```NO```这些 ```Objective-C```保留字是不方便使用的，或者 JSON 中的 key 值不具备良好的可读性等，通过在类中实现此```类方法```可以达到自定义模型和JSON数据的映射关系，值得提出的是其自定义的映射关系，支持多种情况：
- 常规的 特定某一个key
- 深一层的keyPath（即取出更深层次的keyPath所对应的value）
- 一组 key 或者 keyPath

由此，可进一步优化前文所述问题的解决方案，通过一个属性映射多个 key 值即可，实现如下：

```Objective-C
@property (nonatomic, strong) NSNumber *from_time_beg;
+ (nullable NSDictionary<NSString *, id> *)modelCustomPropertyMapper {
    return @{@"from_time_beg": @[@"from_time_beg", @"from_beg_time"]};
}
```
## 延伸
- 第三方框架在使用时更要注重学习其API和原理。
- YYModel 对于属性的数据类型并不强制要求，可以根据实际业务的需要为属性定义类型，针对一些数值类型可能会越界的情形需要另外考虑。
- 注意 YYModel 已经超过一年时间未更新，基于兼容性考虑，仍然有必要对所有的模型对象继承于一个 BaseModel，便于以后的迁移到其他数据转模型方案。

- Xcode 8.2 以后默认编译会对 YYModel 的注释提出警告，可以在 build settings 中搜索 comment 将 Apple LLVM 8.0 - Warnings - All languages -> Documentation Comments 的选项设为 NO，即可消除警告。

![消除注释警告](http://upload-images.jianshu.io/upload_images/73339-6e7f3d00dbbf1cdd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
