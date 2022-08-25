#### 函数嵌套
在 Swift 中，在函数内部定义一个函数即函数嵌套，举例如下：
```Swift
func foo() {
     var a = 1
      func bar() {
             a += 1
      }

     bar()
}
```

在 OC 中没有这类特性，不过如果联想到 Swift 中函数实际是一种有名字的闭包，那么函数嵌套的思想就可以延伸到 OC 语言中了。


#### OC 函数嵌套的实现
```Objective-C
- (void)foo {
     __block NSInteger a = 1;
     void(^bar)(void) = ^{
             a += 1;
      };
      bar();
}
```

由此可见，将嵌套的函数逻辑封装到一个 block，这样就可以在需要时直接调用 block，而不需要另外声明一个方法了。

#### 应用场景的延伸
凡用到嵌套函数的场景，往往也是这一段需要在一个函数/方法的内部多次使用逻辑，不需要外界知晓，比如：
- 地址格式化逻辑
- 日期格式化
- 数据解析操作

这些 block 变量就像一个个小函数一样随时调用，举个应用的例子：
```Objective-C
    NSString *(^formatDate)(NSDate *) = ^NSString *(NSDate *date) {
        NSString *str = [date formatYMD];
        return str ?: @"";
    };
    
    NSString *(^formatStamp)(NSString *) = ^NSString *(NSString *stamp){
        NSDate *date = [NSDate dateWithTimeIntervalSince1970:stamp.doubleValue];
        return formatDate(date);
    };
    
    userInfo = @{
                 kCellLeftTitle : @"起运时间",
                 kCellRightContent : formatStamp(self.model.from_begin_time),
                 kCellModelKey : [NSDate dateWithTimeIntervalSince1970:self.model.from_begin_time.doubleValue],
                 kTextFieldDisableKey : @YES,
                 kHasIndicatorKey : @YES,
                 };
    row.didSelectCellBlock = ^(NSIndexPath *indexPath, id value, id cell) {
        WSDatePickerView *picker = [[WSDatePickerView alloc] initWithDateStyle:DateStyleShowYearMonthDay
                                                                 CompleteBlock:^(NSDate *selected) {
            value[kCellModelKey] = selected;
            value[kCellRightContent] = formatDate(selected);
            [weakSelf.tableView reloadRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationNone];
        }];
        picker.selectedDate = value[kCellModelKey];
        [picker show];
    };
```
上述代码中是在一个方法体的内部，声明了两个 block 变量做嵌套函数使用，分别是 formatDate 和 formatStamp，用于处理模型数据中的 日期和时间戳的字符串表示逻辑。
