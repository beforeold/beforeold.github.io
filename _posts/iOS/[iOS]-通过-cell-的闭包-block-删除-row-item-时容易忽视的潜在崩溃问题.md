## 背景
有些需求会在 UITableViewCell 或者 UICollectionViewCell 上有一个```删除```按钮，用于删除这个 cell，正常的思路是：数据与界面同步，即删除 cell 时同时删除数据源，如下：
```Objective-C
// Cell 的声明
static NSString *const  kIdentifier = @"kIdentifier";

@interface SomeTableViewCell : UITableViewCell

@property (nonatomic, copy) void(^deleteBlock)(SomeTableViewCell *);

@end


// 有问题的Cell 的数据源方法
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    SomeTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:kIdentifier];
    
    __weak typeof(self) weakSelf = self;
    cell.deleteBlock = ^(SomeTableViewCell *theCell){
        [weakSelf.dataArray removeObjectAtIndex:indexPath.row];  // 移除数据源
        [weakSelf.tableView deleteRowsAtIndexPaths:@[indexPath] 
                  withRowAnimation:UITableViewRowAnimationLeft]; // 移除cell
    };
    
    
    return cell;
}
````

> 注意示例代码中的两处 weakSelf 的使用都是必要的，存在潜在的循环引用需要进行手动处理，详见[前文](http://www.jianshu.com/p/b33e5989a352)。

然则，以上示例的处理方式存在潜在的崩溃和业务异常风险。通过 cell 的 block 所捕获的 indexPath，在未删除 cell 的情况下的确对应了 cell 所在的位置，而在按照上述方式删除一个 cell 后，其他 cell 所在的真实 indexPath 已经发生了变化，而 cell 的 block 所捕获的 block 却并未变化，那么进一步删除第二个 cell 时，问题就会显现，即删除第二个 cell 时删除的实际是删除第一个cell 之前所捕获的 indexPath，也就是说，此时第二个 cell 所捕获的 indexPath 已经不代表 这个 cell 。

## 解决方案
既然明确了问题的来源，那么处理方式也很清晰。思路不变，在实现细节上进行优化，根据 cell 本身获取 indexPath 进行 cell 的删除，而不能使用外部所捕获的 indexPath，代码如下:

```Objective-C
// 合理的 Cell 数据源方法
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath_right:(NSIndexPath *)indexPath {
    SomeTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:kIdentifier];
    
    __weak typeof(self) weakSelf = self;
    cell.deleteBlock = ^(SomeTableViewCell *theCell){
        // 切记这里不能使用外部的 tableView 变量，因为 cell 是 tableView 的间接子视图
        // cell 因此被 tableView 强引用
        // 如果在 block 内直接使用 tableView，则 cell 强引用 tableView，造成循环引用
        // 当然也不能直接使用外部的 cell 变量（这个Xcode 会直接给出警告）
        NSIndexPath *rightIndexPath = [weakSelf.tableView indexPathForCell:theCell]; // 获取真实 indexPath
        [weakSelf.dataArray removeObjectAtIndex:rightIndexPath.row]; // 移除真实数据源
        [weakSelf.tableView deleteRowsAtIndexPaths:@[rightIndexPath]
                     withRowAnimation:UITableViewRowAnimationLeft]; // 移除真实 cell
    };
    
    
    return cell;
}
```

## 小结

PS：还需要注意的一点在于 直接 reloadData 是没有问题的，但是会失去一些删除的动画效果，也是不必要的性能损耗。
同样地，在新增一个 cell 时也需要注意。
