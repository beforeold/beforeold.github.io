## 新情况
关于[]下标的用法有新的发现，在[新的文章](http://www.jianshu.com/p/0bb1b6bee194)中进行了补充

## 起因
在调试基于[AFNetworking](https://github.com/AFNetworking/AFNetworking)封装的HTTP请求时的发现更新了对字典的一些错误认识。
场景是用户登录后需要在请求序列化器(`requestSerializer`)的header中设置token对应field的值，
如果登录之前会传入nil，登录后传入服务端返回的token进行后续的请求
AFNetworking这一方法的内部实现如下
```Objective-C
// 可变字典
@property (readwrite, nonatomic, strong) NSMutableDictionary *mutableHTTPRequestHeaders;


- (void)setValue:(NSString *)value forHTTPHeaderField:(NSString *)field
{
	[self.mutableHTTPRequestHeaders setValue:value forKey:field];
}
```
可以看到对mutableHTTPRequestHeaders为可变字典类型，保存value的方式是使用KVC的
而不是可变字典的setOjbect:forKey:方法
因为之前对字典的认识是内部不能有nil对象的，担心会传入nil而导致崩溃`crash`，所以对不可变字典的存值进行了测试

## 测试过程和结果
创建一个新的可变字典，保存一组正常的key-Value
分别使用setObject:forKey:，setValue:forKey:和使用字典[]语法糖传入nil值对之前保存的key进行覆盖测试
结果如下

| 方法 | 结果 |
|:--------:|:---------:|
| setObject:forKey: | 崩溃 |
| setValue:forKey: | 覆盖为nil |
| []语法糖 | 覆盖为nil |

代码如下
```Objective-C
NSMutableDictionary *dic = [NSMutableDictionary dictionary];
    
    [dic setObject:@"object" forKey:@"key"];
    NSLog(@"dic setValue %@", dic);
    
    [dic setValue:nil forKey:@"key"];
    NSLog(@"dic  after KVC %@ ", dic);
    
    
    [dic setObject:@"object" forKey:@"key"];
    NSLog(@"dic setValue %@", dic);
    
    dic[@"key"] = nil;
    NSLog(@"dic after []语法糖 %@", dic);
    
    
    [dic setObject:@"object" forKey:@"key"];
    NSLog(@"dic setValue %@", dic);
    
    [dic setObject:nil forKey:@"key"];
    NSLog(@"dic after setObject %@ ", dic);
```
控制台输出的Log
![控制台Log](http://upload-images.jianshu.io/upload_images/73339-44577b3db1759390.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 小结
实际上setObject:forKey:显式地传入nil时Xcode会给出警告
使用 KVC 或者 [] 语法糖对可变字典进行 key-value 存值可以达到覆盖原有值的目的
因此，根据实际情况需要进行方法的选择
- 如果是有意识地想使用nil来覆盖原值时调用KVC或者[]点语法
- 如果不希望对应的key出现nil值，那么就使用setOjbect:forKey:方法，这样当出现异常时的崩溃可以定位错误

此外，经过 [@李俊峰](http://www.jianshu.com/u/67bf1b98e788)同学的提醒，在 ```NSKeyValueCoding.h``` 中可以看看到 NSMutableDictionary 的 category 中有注释说明，对于可变字典来说 KVC 实际上是调用了可变字典的 setObject: forKey: 方法，当 object 为 nil 时 调用 removeObjectForKey:。
```Objective-C
@interface NSMutableDictionary<KeyType, ObjectType>(NSKeyValueCoding)
NSKeyValueCoding.h
/* Send -setObject:forKey: to the receiver, unless the value is nil, in which case send -removeObjectForKey:.
*/
- (void)setValue:(nullable ObjectType)value forKey:(NSString *)key;

@end
```
