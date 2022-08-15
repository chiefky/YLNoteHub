在`ViewController`中有`viewDidLoad`、`viewWillAppear`等时机，可以根据需要布局，那如果自定义View该如何正确在自己的声明周期布局？

## 一、view初始化和绘制

- `initWithFrame`: 如果是代码实现的view创建，那么使用这个函数进行初始化
- `initWithCoder`: 如果使用Interface Builder创建的view，那么这个函数进行初始化，在这个函数中，view的frame还没有计算
- `drawRect:`: 只有你想自己绘制内容的时候重构这个函数，如果不做任何内容，那就建议不要重构这个函数
- `awakeFromNib`: 如果使用的Interface Builder创建的view，这个函数会调用

这个初始化过程的区别是你使用的是纯代码创建的view还是使用的Interface Builder创建的view

## 二、布局方式的区别

### 2.1、布局方式

现在布局基本都是采用的`Masonry`或者`snpkit`进行动态布局，当然也有部分使用的`frame`布局，这是两种主要的布局方式。

### 2.2、动态布局和AutoresizingMask

`UIView`还有一个`AutoresizingMask`属性，也是用来控制`UIView`的布局的，所以如果使用的autolayout动态布局，那么需要将view的`translatesAutoresizingMaskIntoConstraints`设置为`false`。即可开始通过代码添加Constraint，否则View还是会按照以往的`autoresizingMask`进行计算。

默认情况下如果你是使用的Interface Builder创建view，那么这个值已经设置为false。如果使用的纯代码创建，这个默认值是true。当然如果你是用的`Masonry`或者`snpkit`，库里面再布局时已经设置了这个值为`false`，所以在日常不注意的情况下，可能并没有设置过这个值

### 2.3、不同布局的创建和更新时机

#### 2.3.1、`Masonry`或者`snpkit`动态布局

如果你是使用的动态布局，推荐的方式是在`init`里面进行初始布局，在`updateConstraints`更新布局，如果你的布局限制不需要更改，那么可以不用再`updateConstraints`里面更新布局了

比如

```
    override init(frame: CGRect) {
        super.init(frame: frame)
        self.p_createUI()
    }
    
    func p_createUI() {
        self.addSubview(mView)
        mView.snp.makeConstraints { (make) in
            make.center.equalToSuperview()
            make.width.equalTo(20)
            make.height.equalTo(30)
        }
    }
    
    //如果不需要翻转等重新布局，可以不用实现这个
    override func updateConstraints() {
        mView.snp.updateConstraints { (make) in
            make.height.equalTo(40)
        }
        super.updateConstraints()
    }
```

#### 2.3.2、frame布局

如果你是使用的frame布局，推荐在init初始化布局，在`layoutSubviews`更新布局，如果布局不需要更改，`layoutSubviews`可以不用实现

比如

```
    override init(frame: CGRect) {
        super.init(frame: frame)
        self.p_createUI()
    }
    
    func p_createUI() {
        self.addSubview(mView)
        mView.frame = CGRect(x: 0, y: 0, width: 100, height: 300)
    }
    
    //如果不需要翻转等重新布局，可以不用实现这个
    override func layoutSubviews() {
        super.layoutSubviews()
        mView.frame = CGRect(x: 0, y: 0, width: 100, height: 200)
    }
```

## 三、总结

### 3.1、不要在`updateConstraints`和`layoutSubviews`初始化布局

如果用的是动态布局，在根据位置信息制定好限制之后，其实很少会去需要改变布局，但是`updateConstraints`和`layoutSubviews`会在不同的时机多次调用，如果你在这个里面写了一样的重复布局，需要重复浪费资源去计算，所以视图产生大量冗余更改时，才应该重写该方法，而不应该在这个方法中初始化布局

### 3.2、不要在layoutSubviews写动态布局

如果需要更新布局，需要在`updateConstraints`中实现，而不要在`layoutSubviews`中实现，因为有可能动态布局的约束会更改父view或者view的大小、状态，这样会造成死循环调用，所以推荐在`layoutSubviews`更新的话更新的是frame，而不是view的约束

### 3.3、updateViewConstraints不调用的情况

如果发现VC的`updateViewConstraints`和View`updateConstraints`不调用的情况，可以设置view的`requiresConstraintBasedLayout`

视图并不是主动采用constraint-based的。在非constraint-based的情况下-updateConstraints,可能一次都不会被调用，解决这个问题需要重写该类方法并返回true。

这里要注意，如果一个view或controller是由interface builder初始化的，那么这个实例的`updateViewConstraints`或`updateConstraints`方法便会被系统自动调用，起原因应该就是对应的`requiresConstraintBasedLayout`方法返回true。而纯代码初始化的视图`requiresConstraintBasedLayout`方法默认返回false。

所以在纯代码自定义一个view时，想把约束写在updateConstraints方法中，就一定要重写`requiresConstraintBasedLayout`方法，返回true。

至于纯代码写的`viewController`如何让其`updateViewConstraints`方法被调用。解决办法是手动调用其view的`setNeedsUpdateConstraints`方法。