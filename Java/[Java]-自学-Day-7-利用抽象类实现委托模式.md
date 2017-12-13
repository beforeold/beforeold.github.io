#### 委托模式
委托模式是 iOS 的开发中的经典实践，大意是：
- 一个对象已经具备了一定的功能，但可以对外部对象发出事件或者向外部对象索取数据
- 为了让这个类具备扩展性，这个类不直接依赖外部对象，而只是约定了外部对象应该具备的 API。

在 OC 中定义 API 的方式叫“协议” ```Protocol```，在 Java 中叫“接口” ```Interface```，以熟悉的 iOS 的 TableView 和 ViewController 的关系为例，TableView 可以通过协议要求外部对象 ViewController 提供数据源，或者向 ViewController 发送事件消息，但对 ViewController 的实际类型不得而知，通常称 ViewController 为 TableView 的 delegate/datasource。

前两天学习到 Java 的抽象类，抽象类一个很重要的特性是 声明抽象函数，具体的子类实现时如果不继续做抽象类，则必须实现抽象函数。这个特征具备了约定接口的功能，因此尝试使用 Java 抽象类实现委托模式。

#### 实施方案

第一步：定义 API 抽象类，TableViewDelegate，抽象类，由于表征 TableView 对外发送事件的 API，其中声明了一个函数用于约定 TableView 点击了某一行发出的事件，第一个参数是 TableView 自身，第二个参数是点击的 item 序号。
```Java
public abstract class TableViewDelegate 
{
	protected abstract void tableViewDidSelectItem(TableView tableView, 
                                                    int index);
}
```

第二步：在 TableView 内部事件发生时，调用委托对象 delegate，TableView实现了列表显示的功能（这些代码不会实现），示例代码中提供了一个模拟函数来表示用户点击了 TableView 的那一行的事件，事件发生后调用 TableView的 simulateTouchAtPostion 函数。 在 TableView 的内部有一个成员变量即抽象类引用 TableViewDelegate delegate，当 delegate 不为空时，调用约定的抽象函数，将事件转发给 delegate 对象。
```Java
public class TableView {
	TableViewDelegate delegate;
	public void simulateTouchAtPosition(int position) {
		if (delegate != null) {
			delegate.tableViewDidSelectItem(this, position);
		}
	}
}
```

第三步，利用多态和内部类实现委托事件的最终传达。
需要获知 TableView 点击事件的外部对象其实是 ViewController，粗暴地将
 ViewController 继承 TableViewDelegate 类，再将 ViewController 赋值作为
 TableView 的 delegate，委托模式已经实现。
```Java
public class ViewController extends TableViewDelegate {
        TableView tableView
    	void tableViewDidSelectItem(TableView tableView, int index) {
		System.out.println("get " + tableView + " at " + index + " .");
    }
}

public static void main(String[] args) {
    ViewController vc = new ViewController();
    vc.tableView = new TableView()
    vc.tableView.delegate = vc;

    vc.tableView.simulateTouchAtPosition(1024);   // 模拟用户点击
}
```

但 ViewController 作为视图管理的类，不能随随便便就继承TableViewDelegate，做一个继承自 TableViewDelegate 的内部类对象作为缓冲，更加妥当：
- 定义 ViewController 的内部类 TrueDelegate，继承自 TableViewDelegate
- TrueDelegate 实现了父类抽象类的抽象函数, didSelect.. 函数
- 当 TableView 被用户触发时，调用其 Delegate（此时实际是 TrueDelegate 对象，属于继承的多态特性）
- TrueDelegate 有一个成员变量引用到 外部的实际需要委托事件的 ViewController 对象
- TrueDelegate 作为内部类可以方便地调用 ViewController 的函数通知ViewController 其受到的事件和参数

```Java
class ViewController {
    class TrueDelegate extends TableViewDelegate {
    	ViewController vc;
        public void tableViewDidSelectItem(TableView tableView, int index) {
    		vc.tableViewDidSelectItem(tableView, index);
    	}
    }

	TableView tableView;
	TrueDelegate delegateObject;
    ViewController() {
    	tableView = new TableView();
        delegateObject = this.new TrueDelegate();
        delegateObject.vc = this;
        
        tableView.delegate = (TableViewDelegate)delegateObject;
    }

	void tableViewDidSelectItem(TableView tableView, int index) {
		System.out.println("get " + tableView + " at " + index + " .");
    }

    public static void main(String[] args) {
        ViewController vc = new ViewController();

        vc.tableView.simulateTouchAtPosition(1024);
    }
}
```
>为什么是内部类，因为内部类对所在的外部类 API 更便于调用。

综上可见如下引用关系：
```Java
TableView -> TableViewDelegate
ViewController <-> TrueDelegate
ViewController -> TableView
```
通过抽象类的子类 TrueDelegate 过桥转发事件，实现委托模式，在 TableView 不依赖 ViewController 的情况下顺利通知了 ViewController，这样一来，TableView 通过抽象类定义了 API，具备让任何其他使用 TableView 的外部对象做委托的特性。

> PS： 这几天学习还是着急了，不敲代码是不行的，调整一下学习计划：
>- 减少每日学习内容 
>- 每日必须完成练习。

#### update 2017-10-20
前文中对内部类的使用十分粗糙
- 可以进行一些语法的优化，比如内部对象是可以直接调用外部对象函数或成员等
- 在后面的学习中[内部类和匿名内部类](http://www.jianshu.com/p/40940eef66bc)，可以看出委托模式可以结合匿名内部类和接口进行非常流畅的设计

#### 欢迎简书沟通交流。
