## AutoresizingMask和AutoLayout

### AutoresizingMask

我们在使用AutoResizing进行布局的时候，其主要思想就是设置子视图跟随父视图的frame变化而变化。具体的情况，我们可以设置左跟随，右跟随等等.

```objective-c
typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {
    UIViewAutoresizingNone                 = 0,
    UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,
    UIViewAutoresizingFlexibleWidth        = 1 << 1,
    UIViewAutoresizingFlexibleRightMargin  = 1 << 2,
    UIViewAutoresizingFlexibleTopMargin    = 1 << 3,
    UIViewAutoresizingFlexibleHeight       = 1 << 4,
    UIViewAutoresizingFlexibleBottomMargin = 1 << 5
};
```

### AutoLayout

AutoLayout相比AutoResizing更加实用，是可以完全替代AutoResizing的一种自动布局方式。而在使用AutoLayout前，我们必须理解一个属性，那就是translatesAutoresizingMaskIntoConstraints。该属性表示autoresizingMask和autolayout两种方式的转换。这个属性对于在代码中生成的view来说默认是true，而对于IB中拖出来的view来说默认是false.

 对于这一属性，官方文档给出的解释是这样的：

```javascript
     /* By default, the autoresizing mask on a view gives rise to constraints that fully determine
 the view's position. This allows the auto layout system to track the frames of views whose
 layout is controlled manually (through -setFrame:, for example).
 When you elect to position the view using auto layout by adding your own constraints,
 you must set this property to NO. IB will do this for you.
    */
```

从以上的描述中，我们可以知道在使用AutoResizing布局时，AutoLayout会根据autoResizing来创建同等行为的constraint出来确定视图的位置。从而实现了视图的自动布局。而当我们确定选择使用AutoLayout添加自己的约束的时候，我们必须设置此属性为NO，XIB中这个属性默认是NO。 在实际的使用过程中，我还需要注意两点： 1.当我们设置这个属性为YES的时候，view的布局结果由AutoResizingMask，frame,center这些因素共同决定，如果再在其上添加AutoLayout约束，自定义的AutoLayout约束就会和AutoResizing里Autolayout约束冲突而报错。 2.我们设置该属性为NO，AutoResizing并不会直接失效，只有当我们为视图设置了constraint之后，AutoResizing才会失效。

那么AutoLayout在开发中具体如何使用呢，这其实分为两种情况，一种是借助xib中的约束功能通过连线的方法实现。还有一种就是代码直接添加约束，但是代码自动布局是一件很麻烦的事，我们通常又会借助第三方即Masonry，下章节将会简要介绍Masonry的用法。