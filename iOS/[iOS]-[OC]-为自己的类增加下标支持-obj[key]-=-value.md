## 楔子
本文是前面一篇文章[《[iOS] [OC] 可变字典下标[]语法糖不是setObject:forKey: 而等效于KVC》](http://www.jianshu.com/p/2aebcff92a2e)探索的新发现。

## 下标的原理
下标subscripting是OC开发中常用的字典和数组的取值和设值方式，其中不可变字典和数组可以取值，可变字典和数组通过继承可以取值，同时还支持设值
```Objective-C
    NSDictionary *dic = @{@"key":@"value"};
    NSString *value = dic[@"key"];  // 不可变字典取值
    
    NSMutableDictionary *mutableDic = [NSMutableDictionary dictionary];
    mutableDic[@"key"] = @"value";  // 可变字典设值
    NSString *mValue = mutableDic[@"key"]; // 可变字典取值
    
    
    NSArray *array = @[@1, @2];
    NSNumber *num = array[0]; // 不可变数组取值
    
    NSMutableArray *mutableArray = [NSMutableArray array];
    [mutableArray addObject:@1];
    mutableArray[0] = @2;   // 可变数组设值
    NSNumber *mNum = mutableArray[0];  // 可变数组取值
```
下标的使用是iOS6以后支持的，取值和设值的原理是编译器调用了一套非正式的协议`informal-protocol`，这套协议的[文档](http://clang.llvm.org/docs/ObjectiveCLiterals.html)将下标分为两类，字典样式`dictionary-style`和数组样式`array-style`，分别要求实现对应协议的方法，从而支持下标的使用
```Objective-C
// 字典样式
- (nullable ObjectType)objectForKeyedSubscript:(KeyType)key; // 取值
- (void)setObject:(nullable ObjectType)obj forKeyedSubscript:(KeyType <NSCopying>)key; // 设值

// 数组样式
- (ObjectType)objectAtIndexedSubscript:(NSUInteger)idx ; // 取值
- (void)setObject:(ObjectType)obj atIndexedSubscript:(NSUInteger)idx ; // 设值
```
除了实现实际需求的方法外，还需要定义的类在显示的声明或者遵从包含上述需求的特定方法的协议，
从而能被编译器正确识别。

此时回过头去看苹果的`NSArray`、`NSMutableArray`、`NSDictionary`、`NSMutableDictionay`在头文件有方法的声明，以及`JSValue`、`JSContext`都有`SubscriptSupport`的`category`，以及第三方的比如谷歌`Protobuffer`的`GPBDicionary`类，数据库管理`FMDB`的`FMResultSet`类以及钥匙串管理类`UICKeyChainStore`都有相关的声明和实现。

## 小结
如此以来就可以解释`mutableDic[key] = nil`不会出错的原因了，因为使用的是下标方法，而根据apple的可变字典文档的说明，传入`nil`会移除对应`key`的`value`。

文档是一座宝库。
