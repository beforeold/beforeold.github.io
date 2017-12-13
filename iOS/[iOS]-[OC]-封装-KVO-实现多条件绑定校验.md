#### 背景
在某些场景下，需要多个条件同时满足后才能进行下一步操作，比如登录页面账号、密码的输入，随着内容格式是否满足要求，登录按钮动态变化为是否可点击 ```enabled```。这种情况下常见的解决办法是：通过 ```UITextField``` 的 ```Target-Action``` ，对两个 ```textField``` 的输入变化 ```Editing-Change```事件进行监听，再在一个方法中统一判断改变按钮状态，举例如下：

```Objective-C
/// 输入框内容变化事件
- (IBAction)userNameOrPasswordChanged:(UITextField *)field {
        [self refreshLoginButton];
}
/// 刷新登录按钮的可点击状态
- (void)refreshLoginButton {
        BOOL isUserNameOK = [self.userNameField.text isNameOK....];
        BOOL isPasswordOK = [self.passwordField.text isPasswordOK....];

        self.loginButton.enable = isUserNamedOK && isPasswordOK;
}

```
如果出现下面的情况时，处理变得更为复杂：
- ```TextField``` 在 ```UITableViewCell``` 上，这意味着获取输入框信息的需要事先在 cell 上标记为属性，或者写死 ```Cell``` 所在的 ```indexPath```
- 当出现超过 **3** 个的校验时，多个校验逻辑写在一些，导致耦合增加，不便维护和拆分

联系到输入的结果往往也用于请求的参数，注入的请求 ```model``` 或者字典 ```key-value``` 中，那么可借助 ```KVO``` 来监听输入变化后进行 ```model``` 注入的变化，

```Objective-C
@implementation ViewController

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    static NSString *identifier = @"XCell";
    XCell *cell = [tableView dequeueReusableCellWithIdentifier:identifier];
    if (!cell) {
        cell = [[XCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:identifier];
    }
    
    __weak typeof(self) weakSelf = self;
    cell.textChangeBlock = ^(NSString *text, XCell *cell) {
        if (indexPath.row == 0) {
            weakSelf.model.userName = text;
        } else {
            weakSelf.model.password = text;
        }
    };
    
    return cell;
}

// ..... cliped
```
> PS：如果能对每一行的数据和展示做进一步的分离，则耦合更小，可以参考旧文[《[iOS] [OC] 轻量级的表单框架 GSForm（附demo）》](http://www.jianshu.com/p/b4c834976590)的处理。

再结合 ```KVO``` 对 ```model``` 的属性进行监听即可实现不同字段的校验。不过 ```KVO``` 的 ```API``` 是众所周知的不便利：
- 需要添加观察者后
- 再在另外一处实现非正式协议完成观察者的回调
- 还需要在合适的时候移除观察者

因此针对性地进行封装是必要的。

#### 介绍 KVOValidator
针对多个条件的校验封装了 ```GSKVOValidator```，类似于通知中心 ```usingBlock``` 处理通知的监听（[可参考前文第四节](http://www.jianshu.com/p/4b2dff2bbdb4)），使用 ```block``` 监听 ```KVO``` 的变化。原理是：
- 另起一个对象，充当监听者
- 并在监听发生变化时通过 ```block```或者代理进行外部回调
- 当这个对象销毁时 ```dealloc``` 时自动移除 KVO 的监听，不再需要手动移除

这个对象称为 KVOAction，实现如下：

```Objective-C
- (instancetype)initWithObservee:(id)observee
                      capturable:(BOOL)capturable
                         keyPath:(NSString *)keyPath
                       validator:(BOOL (^)(GSKVOAction<id> *action))validator 
{
    self = [super init];
    if (self) {
        NSParameterAssert(observee);
        NSParameterAssert(keyPath);
        
        if (capturable) {
            _observee_retain = observee;
        } else {
            _observee_assign = observee;
        }
        
        _capturable = capturable;
        _keyPath = [keyPath copy];
        _validator = [validator copy];
        
        [self.observee addObserver:self forKeyPath:_keyPath options:NSKeyValueObservingOptionNew context:NULL];
    }
    
    return self;
}
```

在 ```KVO``` 发生变化回调时，```Action``` 对象回调保存的 ```block```，如下：

```Objective-C
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSKeyValueChangeKey,id> *)change
                       context:(void *)context
{
    if (![keyPath isEqualToString:self.keyPath]) return;
    
    if (self.internalDelegate) {
        [self.internalDelegate kvoActionDidChange:self];
        
    } else if (self.validator) {
        self.validator(self);
    }
}

- (void)dealloc {
    [self removeKVO];
}

```

封装 ```KVOAction``` 时，值得留意的是因为 ```Action``` 对象同样是需要被一个对象持有来避免立即释放的，而 ```Action``` 要引用被观察者，这时，就需要避免循环引用的发生，如果被观察者同时是 ```Action``` 的持有者，那么 ```Action``` 则不能对被观察者进行强引用，否则则可进行强引用。 对于不能强引用 ```strong``` 的情况，也不能使用弱引用 ```weak```，原因是 ```Action``` 在释放 ```dealloc``` 时仍旧需要移除 ```KVO```，这时 ```weak``` 引用达不到移除的目的，使用 ```assign``` 或者 ```unsafe_unretained``` 是合适的，其他情况下则使用 ```strong``` 被 ```Action``` 强应用是合适的。

进一步地，为了达到多个 ```Action``` 联合工作，引入 ```KVOValidator```：
- 要求 ```KVOAction``` 的 ```block``` 返回一个 ```BOOL``` 值
- ```KVOValidator``` 持有多个 ```Action```
- 当任意一个 ```Action``` 触发 ```KVO``` 时，```Action``` 将通过内部代理 ```internalDelegate``` 转发给 ```KVOValidator```
- ```Validator``` 遍历所有 ```Action``` 统一判断所有的 ```Action``` 校验是否合格
- 最终将校验结果通过 ```result_block``` 进行回调


```Objective-C
/// 持有所有 Action 并成为 Action 的代理
- (instancetype)initWithActions:(NSArray<GSKVOAction *> *)actions
                    allValidate:(BOOL (^)(NSArray <GSKVOAction *> *))allValidate
                         result:(void (^)(BOOL, NSArray *, GSKVOAction *))result {
    self = [super init];
    if (self) {
        
        _actions = [actions copy];
        _allValidate = [allValidate copy];
        _result = [result copy];
        
        [_actions makeObjectsPerformSelector:@selector(setInternalDelegate:) withObject:self];
    }
    
    return self;
}

#pragma mark - protocol
/// Action 的代理通过内部私有的代理进行回调
- (void)kvoActionDidChange:(GSKVOAction *)action {
    if (action.validator && !action.validator(action)) {
        [self handleResult:NO failed:action];
        return;
    }
    
    BOOL validateOK = YES;
    GSKVOAction *failed = nil;
    for (GSKVOAction *item in self.actions) {
        if (!item.validator) continue;
        if (item == action) continue;
        
        validateOK &= item.validator(item);
        if (!validateOK) {
            failed = item;
            [self handleResult:NO failed:failed];
            return;
        }
    }
    
    if (self.allValidate) {
        validateOK &= self.allValidate(self.actions);
    }
    
    [self handleResult:validateOK failed:failed];
}

/// 执行结果的回调
- (BOOL)handleResult:(BOOL)ret failed:(GSKVOAction *)failed {
    _isRecentValid = ret;
    
    !self.result ?: self.result(ret, self.actions, failed);
    
    return _isRecentValid;
}
```

#### 源代码
[GitHub](https://github.com/beforeold/GSKVOValidator)

#### 小结
- 封装的过程中，需要着重考虑到对象的引用关系，以及在 ```Action``` 释放时及时移除 ```KVO``` 的处理的安全性。
- 内部通过协议的形式做进一步的统一处理，实现多个条件的统一校验。
- 可以进一步的扩展，将固定的返回一个 校验结果 BOOL 值的策略调整为传入一个运算 ```block```，达到一个 ```reduce``` 的效果。（注： ```reduce``` 是将一组结果通过指定的运算后变成一个结果的函数式编程思想。）

- ```Facebook``` 有一套完善的 ```KVO``` 封装，[KVOController](https://github.com/facebook/KVOController) 值得进一步挖掘。
