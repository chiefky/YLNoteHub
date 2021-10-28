# 个人笔记-关联对象、weak对象实现原理

前言

 [了解什么是哈希表、hash数组](https://juejin.cn/post/6844903778940878861#heading-10)

SideTables: 哈希数组 (个人理解为数组下标是通过向某一个哈希函数传入key得到，然后从数组中直接取值)

weak_table：哈希表（也称散列表， 哈希表本质是一个数组，数组中的每一个元素成为一个箱子，箱子中存放的是键值对）

### 1. weak对象底层实现原理

**底层数据模型：hash表+数组**

散列表（hash表）【key: 弱引用对象的地址，value：weak指针的地址数组】

如图：

<img src="../images/weak底层类图.png" alt="img" style="zoom:80%;" />





<img src="../images/weak_table存储结构.png" alt="img" style="zoom:100%;" />

```objective-c
Person *object = [Person alloc];
id __weak objc = object;
```

创建了一个NSObject对象`obj`，然后用`weakObj`对obj做弱引用。

**weak 实现原理的概括**

Runtime维护了一个weak表，用于存储指向某个对象的所有weak指针。weak表其实是一个hash（哈希）表，Key是所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象的地址）数组。

weak 的实现原理可以概括一下三步：

1、初始化时：runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。

2、添加引用时：objc_initWeak函数会调用 objc_storeWeak() 函数， objc_storeWeak() 的作用是更新指针指向，创建对应的弱引用表。

3、释放时，调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。

一个对象的释放过程：



### 2.关联对象的实现原理：

**底层实现使用的数据模型：hash表+hash表**

![](./images/association存储结构.png)

![](./images/association map.png)

**Association实现原理的概括**

以此为例：

```objective-c
- (void)setAssociatedObject:(NSArray *)associatedObject {
 objc_setAssociatedObject(self,@selector(associatedObject),associatedObject,OBJC_ASSOCIATION_COPY_NONATOMIC);
}

```



关联对象的实现原理需要从两方面讲：

* 关联对象的添加

* 关联对象的删除

  **关联对象的添加**过程

  > 1. disguised（包装，伪装）被关联对象
  >
  > 2. 根据（policy,value)构建出新关联对象---association
  >
  > 3. 根据policy对关联对象进行retain或copy操作
  >
  >    ```c++
  >    inline void acquireValue() {
  >      if (_value) {
  >          switch (_policy & 0xFF) {
  >          case OBJC_ASSOCIATION_SETTER_RETAIN:
  >              _value = objc_retain(_value);
  >              break;
  >          case OBJC_ASSOCIATION_SETTER_COPY:
  >              _value = ((id(*)(id, SEL))objc_msgSend)(_value, @selector(copy));
  >              break;
  >          }
  >      }
  >    }
  >    ```
  >
  > 4. 获取AssociationsManager对象manager（实质就是一个加锁操作），通过manager取到**全局的哈希Map（AssociationsHashMap）**------【**被关联对象表**】`<key: diguised(obj),value: ObjectAssociationMap>`
  >
  > 5. 根据value是否为nil分别处理关联逻辑
  >
  >    a. _value != nil： 
  >
  >    第一步：
  >
  >    尝试以diguised(obj)为key向【**被关联对象表**】表插入一个空的**【关联对象表】** ObjectAssociationMap`(<key,关联对象>)`，如果是：
  >
  >    第一次关联对象：插入mapBucketT`<0,0>`，并返回插入结果<iterator, true>，后面以此为依据在锁外将被关联对象的关联标记位置为true；
  >
  >    非第一次关联对象：停止插入，返回结果<iterator, false>；
  >
  >    第二步：
  >
  >    获取第一步中的mapBucketT，尝试向mapBucketT中插入objBucketT`<@selector(associatedObject),ObjectAssociation>` ;
  >
  >    mapBucketT 中不存在key：插入并返回result = <iterator, true>;
  >
  >    mapBucketT 中已存在key：停止插入，返回result = <iterator, false>，根据result=false，将【关联对象表】中对应key关联的旧对象与新对象association进行值交换；然后将旧的值释放；
  >
  >    b. _value == nil:  **等同于将已存在的关联对象置为nil，即移除已存在的关联对象**；
  >
  >    通过【**被关联对象表**】，key<disguised(obj)>找到对应的**【关联对象表】**,再通过【关联对象表】,key找到对应的旧关联对象，与新关联对象进行值交换；（注：这里新关联对象的value==nil）
  >
  >    然后从关联对象表中擦除旧关联对像键值对；如果擦除键值对之后，关联对象表size ==0，就从被关联对象表中把此关联对象表键值对一并擦除。
  >
  >    ```objc
  >    
  >      DisguisedPtr<objc_object> disguised{(objc_object *)object};
  >      ObjcAssociation association{policy, value};
  >    
  >      // retain the new value (if any) outside the lock.
  >      association.acquireValue();
  >      bool isFirstAssociation = false;
  >      {
  >          AssociationsManager manager;
  >          AssociationsHashMap &associations(manager.get());
  >          if (value) {
  >              auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
  >              if (refs_result.second) {
  >                  /* it's the first association we make */
  >                  isFirstAssociation = true;
  >              }
  >              /* establish or replace the association */
  >              auto &refs = refs_result.first->second;
  >              auto result = refs.try_emplace(key, std::move(association));
  >              if (!result.second) {
  >                  association.swap(result.first->second);
  >              }
  >          } else {
  >              auto refs_it = associations.find(disguised);
  >              if (refs_it != associations.end()) {
  >                  auto &refs = refs_it->second;
  >                  auto it = refs.find(key);
  >                  if (it != refs.end()) {
  >                      association.swap(it->second);
  >                      refs.erase(it);
  >                      if (refs.size() == 0) {
  >                          associations.erase(refs_it);
  >    
  >                      }
  >                  }
  >              }
  >          }
  >      }
  >    
  >      // Call setHasAssociatedObjects outside the lock, since this
  >      // will call the object's _noteAssociatedObjects method if it
  >      // has one, and this may trigger +initialize which might do
  >      // arbitrary stuff, including setting more associated objects.
  >      if (isFirstAssociation)
  >          object->setHasAssociatedObjects();
  >    
  >      // release the old value (outside of the lock).
  >      association.releaseHeldValue();
  >    ```

  关联对象删除过程：






