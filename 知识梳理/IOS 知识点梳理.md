---
typora-root-url: ./资源/图片
typora-copy-images-to: ./资源/图片
---

# IOS 知识点梳理

[TOC]

## UI相关

### UITableView

#### 数据源同步问题

1. 并发访问，数据拷贝
2. 串行访问

### UI事件传递以及响应

#### UIView 和 CALayer关系

1. UIView为CALayer 提供内容以及负责处理触摸等事件，参与相应链。
2. CALayer负责显示
3. 体现了单一职责原则

#### 事件传递与视图响应链

1. 事件传递相关方法
``` typ-c

// recursively calls -pointInside:withEvent:. point is in the receiver's coordinate system
// 递归调用- pointInside：withEvent： 方法 判断point是否位于receiver 的坐标系中 决定返回一个receiver
 - (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;

// default returns YES if point is in bounds
- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;
```

2. 事件传递
   * 事件响应
   * `-hitTest:withEvent`
``` flow
st=>start: 点击屏幕
e=>end
window=>operation: UIApplication->UIWindow
hitTest=>operation: -hitTest:withEvent
op3=>operation: Subviews.倒序遍历
op4=>operation: return view
pointInside=>operation: -pointInside:withEvent:
st->window->hitTest->pointInside->op3
```

```flow
st=>start: Start
op1=>operation: sv=[v.subviews[i] hitTest:point withEvent:event];
op2=>operation: return v;
op3=>operation: return sv;
op4=>operation: return nil;
condition1=>condition: !v.hidden &&
v.userInteractionEnabled &&
v.alpha>0.01
condition2=>condition: [v pointInside:point 
withEvent:event];
condition3=>condition: for(int i= v.subviews.count-1;i>=0;i--)
condition4=>condition: sv!=nil
e=>end: Stop
st->condition1
condition1(yes)->condition2
condition1(no)->op4
condition2(no)->op4
condition2(yes)->condition3
condition3(yes)->op1->condition4
condition3(no)->op2
condition4(no)->condition3
condition4(yes)->op3->e
```

3. 视图事件响应方法

   ```objc
   
   // Generally, all responders which do custom touch handling should override all four of these methods.
   // Your responder will receive either touchesEnded:withEvent: or touchesCancelled:withEvent: for each
   // touch it is handling (those touches it received in touchesBegan:withEvent:).
   // *** You must handle cancelled touches to ensure correct behavior in your application.  Failure to
   // do so is very likely to lead to incorrect behavior or crashes.
   - (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
   - (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
   - (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
   - (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
   - (void)touchesEstimatedPropertiesUpdated:(NSSet<UITouch *> *)touches NS_AVAILABLE_IOS(9_1);
   ```



### 图像显示原理

``` sequence
Title:图像显示过程
CPU->总线: 布局 绘图 图片编解码 提交位图 \n(core animation)
总线->GPU: 位图
GPU->总线: 渲染管线 定点着色 图元装配 \n 片段着色 片段处理 像素点 提交到帧缓冲区 frame buffer\n 视频控制器提取(vsync 信号来之前) 显示到屏幕
```

### UI卡顿掉帧的原因

Vsync 16.7ms CPU GPU 在这时间内没有完成task



### UIView 的绘制原理

```flow
st=>start: [UIView setNeedsDisplay]
op1=>operation: [view.layer setNeedsDisplay](相当于打上标记)
op2=>operation: [CALayer display](在当前runloop快要结束时候执行)
condi=>condition: [layer.delegate respondsTo
@selector(displayLayer)]
op3=>operation: 异步绘制入口
op4=>operation: 系统绘制流程
end=>end: 结束
st->op1->op2->condi
condi(yes)->op3->end
condi(no)->op4->end

```

#### 系统绘制

```flow
st=>start: CALayer creates backing store(CGContextRef)
op1=>operation: [CALayer drawInContext]
op2=>operation: [layer.delegate drawLayer:inContext:]
op3=>operation: [UIView drawRect:]
op3=>operation: CALayer uploads backing store to GPU
op4=>operation: [UIView drawRect:]
end=>end: 结束
condi=>condition: layer has a delegate?
st->condi
condi(yes)->op2->op4->op3->end
condi(no)->op1->op3->end
```



CALayer creates backing store (CGContextRef)  如果实现drawInContext：`delegate drawLayer:inContext: ->[UIView drawRect:]`最后提交到GPU CALayer uploads backing store to GPU 



#### 异步绘制

1. [layer.delegate display layer]

- 代理负责生成对应的bitmap 
- 设置该bitmap作为layer.contents属性的值

``` sequence
Title:异步绘制
participant main
participant global queue

Note over main : -[AsyncDrwingView setNeedsDisplay]
Note over main : -[CALayer display] \n -[AsyncDrawingView displayLayer:]
main->global queue: 
Note over global queue : - CGBitmapContextCreate\n -CoreGraphic API \n -CGBitmapContextCreateImage
note over main : other works
global queue->main :
Note over main : -[CALayer setContents:]






```

#### 离屏渲染

1. 概念
   - On-Screen Rendering  意为当前屏幕渲染,指的是GPU的渲染操作是在当前用于显示的屏幕缓冲区中进行
   - Off-Screen Rendering 意为离屏渲染,指的是GPU在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作
2. 何时会触发
   - 圆角 (当和maskToBounds一起使用时)
   - 图层蒙版
   - 阴影
   - 光栅化

3. 为何要避免?
   -  触发离屏渲染的时候会增加GPU工作量,很有可能导致CPU,GPU工作总耗时超过16.7ms 就会造成卡顿掉帧
   - 创建新的渲染缓冲区,内存开销, 多通道渲染管线上下文切换增加GPU工作量

### UI视图相关题目

- 系统的UI事件传递机制是怎样的?(hitTest: pointWithEvent)
- 使UITableView滚动更流畅的方案或思路都有哪些?(从CPU,GPU两方面)
- 什么是离屏渲染
- UIView和CALayer之间的关系是怎样的?



## Objective-C 语言特性

### 分类

1. 你用分类都做了那些事

   - 声明私有方法
   - 声明私有方法
   - 把Framework的私有方法公开

2. 特点

   - 运行时决议 (运行时通过runtime 把分类内容添加到宿主类中)
   - 可以为系统类添加分类

3. 分类中都可以添加哪些内容

   - 实例方法
   - 类方法
   - 协议
   - 属性 (在分类中声明属性 只有get set方法 并没有添加实例变量,如果想添加实例变量就要添加关联对象)

4. 分类结构体

   ``` c++
   //objc-runtime-680版本
   struct category_t{
       const char *name;//名称
       classref_t cls;//所属宿主类
       struct method_list_t *instanceMethods;//实例方法
       struct method_list_t *classMethods;//类方法
       struct method_list_t *protocols;//协议
       struct property_list_t *instanceProperties;//实例变量
       
       method_list_t *methodsForMeta(bool isMeta){
           if (isMeta) return classMethods;
           else return instanceMethods;
       }
       
       property_list_t *propertiesForMeta(bool isMeta){
           if (isMeta) return nil;
           else return instanceProperties;
       }
   }
   ```

5. 加载调用栈

   ``` flow
   op0=>start: _objc_init
   op1=>operation: map_2_images (images 指镜像)
   op2=>operation: map_images_nolock
   op3=>operation: _read_images (读取镜像,加载可执行文件到内存当中)
   op4=>operation: remethodizeClass
   op0->op1->op2->op3->op4
   ```

6. 源码分析

   ``` c++
   /***********************************************************************
   * remethodizeClass
   * Attach outstanding categories to an existing class.
   * Fixes up cls's method list, protocol list, and property list.
   * Updates method caches for cls and its subclasses.
   * Locking: runtimeLock must be held by the caller
   **********************************************************************/
   static void remethodizeClass(Class cls)
   {
       category_list *cats;
       bool isMeta;
   
       runtimeLock.assertLocked();
   
       isMeta = cls->isMetaClass();
   
       // Re-methodizing: check for more categories
       //获取宿主类cls所有还未拼接整合的分类
       if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
           if (PrintConnecting) {
               _objc_inform("CLASS: attaching categories to class '%s' %s", 
                            cls->nameForLogging(), isMeta ? "(meta)" : "");
           }
           //将分类cat拼接到cls
           attachCategories(cls, cats, true /*flush caches*/);        
           free(cats);
       }
   }
   
   /***********************************************************************
   * unattachedCategoriesForClass
   * Returns the list of unattached categories for a class, and 
   * deletes them from the list. 
   * The result must be freed by the caller. 
   * Locking: runtimeLock must be held by the caller.
   **********************************************************************/
   static category_list *
   unattachedCategoriesForClass(Class cls, bool realizing)
   {
       runtimeLock.assertLocked();
       return (category_list *)NXMapRemove(unattachedCategories(), cls);
   }
   
   // Attach method lists and properties and protocols from categories to a class.
   // Assumes the categories in cats are all loaded and sorted by load order, 
   // oldest categories first.
   static void 
   attachCategories(Class cls, category_list *cats, bool flush_caches)
   {
       if (!cats) return;
       if (PrintReplacedMethods) printReplacements(cls, cats);
   
       bool isMeta = cls->isMetaClass();
   
       // fixme rearrange to remove these intermediate allocations
       // [[method_t,method_t,...],[method_t],[method_t,method_t],...]
       method_list_t **mlists = (method_list_t **)
           malloc(cats->count * sizeof(*mlists));
       property_list_t **proplists = (property_list_t **)
           malloc(cats->count * sizeof(*proplists));
       protocol_list_t **protolists = (protocol_list_t **)
           malloc(cats->count * sizeof(*protolists));
   
       // Count backwards through cats to get newest categories first
       int mcount = 0;
       int propcount = 0;
       int protocount = 0;
       int i = cats->count;//宿主类分类的总数
       bool fromBundle = NO;
       while (i--) {//这里遍历是倒序,最先访问最后编译的分类
           auto& entry = cats->list[i];//取到最后一个结构体 其中含有分类 
           /*
           struct locstamped_category_t {
                 category_t *cat;
                 struct header_info *hi;
            };
   
          struct locstamped_category_list_t {
                 uint32_t count;
                 #if __LP64__
                 uint32_t reserved;
                 #endif
                 locstamped_category_t list[0];
           };*/
   
   
           method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
           if (mlist) {
               mlists[mcount++] = mlist;
               fromBundle |= entry.hi->isBundle();
           }
   
           property_list_t *proplist = 
               entry.cat->propertiesForMeta(isMeta, entry.hi);
           if (proplist) {
               proplists[propcount++] = proplist;
           }
   
           protocol_list_t *protolist = entry.cat->protocols;
           if (protolist) {
               protolists[protocount++] = protolist;
           }
       }
   
       auto rw = cls->data();//获取宿主类当中的rw数据,其中包含宿主类的方法列表信息
   
       //主要针对 分类中有关于内存管理相关方法情况下的一些特殊处理
       prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
       /*
       rw 代表类
       methods 代表类的方法列表
       attachLists 方法的含义是 将含有mcount个元素的mlists拼接到rw的methods上
       */
       rw->methods.attachLists(mlists, mcount);
       free(mlists);
       if (flush_caches  &&  mcount > 0) flushCaches(cls);
   
       rw->properties.attachLists(proplists, propcount);
       free(proplists);
   
       rw->protocols.attachLists(protolists, protocount);
       free(protolists);
   }
   
   /*
     addedLists 传递过来的二维数组
     [[method_t,method_t,...],[method_t],[method_t,method_t,method_t],...]
     ------------------------ ---------  ---------------------------------
     分类A中的方法列表(A)          B                C
     addedCount = 3
   */
   
   void attachLists(List* const * addedLists, uint32_t addedCount) {
           if (addedCount == 0) return;
   
           if (hasArray()) {
               // many lists -> many lists
               uint32_t oldCount = array()->count;
               uint32_t newCount = oldCount + addedCount;
               //根据新总数重新分配内存
               setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
               //重新设置元素总数
               array()->count = newCount;
               //内存移动 [[],[],[],[原有的第一个元素],[原有的第二个元素]] 偏移addedCount
               memmove(array()->lists + addedCount, array()->lists, 
                       oldCount * sizeof(array()->lists[0]));
               //内存拷贝 [A,B,C,[原有的第一个元素],[原有的第二个元素]] 这也是分类方法会覆盖宿主类方法的原因
               memcpy(array()->lists, addedLists, 
                      addedCount * sizeof(array()->lists[0]));
           }
           else if (!list  &&  addedCount == 1) {
               // 0 lists -> 1 list
               //如果 原来没list 而且新list只有1个 就直接替换
               list = addedLists[0];
           } 
           else {
               // 1 list -> many lists
               //直接把旧的放在最后
               List* oldList = list;
               uint32_t oldCount = oldList ? 1 : 0;
               uint32_t newCount = oldCount + addedCount;
               setArray((array_t *)malloc(array_t::byteSize(newCount)));
               array()->count = newCount;
               if (oldList) array()->lists[addedCount] = oldList;
               memcpy(array()->lists, addedLists, 
                      addedCount * sizeof(array()->lists[0]));
           }
       }
   
   
   /***********************************************************************
   * methodizeClass
   * Fixes up cls's method list, protocol list, and property list.
   * Attaches any outstanding categories.
   * Locking: runtimeLock must be held by the caller
   **********************************************************************/
   static void methodizeClass(Class cls)
   {
       runtimeLock.assertLocked();
   
       bool isMeta = cls->isMetaClass();
       auto rw = cls->data();
       auto ro = rw->ro;
   
       // Methodizing for the first time
       if (PrintConnecting) {
           _objc_inform("CLASS: methodizing class '%s' %s", 
                        cls->nameForLogging(), isMeta ? "(meta)" : "");
       }
   
       // Install methods and properties that the class implements itself.
       method_list_t *list = ro->baseMethods();
       if (list) {
           prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
           rw->methods.attachLists(&list, 1);
       }
   
       property_list_t *proplist = ro->baseProperties;
       if (proplist) {
           rw->properties.attachLists(&proplist, 1);
       }
   
       protocol_list_t *protolist = ro->baseProtocols;
       if (protolist) {
           rw->protocols.attachLists(&protolist, 1);
       }
   
       // Root classes get bonus method implementations if they don't have 
       // them already. These apply before category replacements.
       if (cls->isRootMetaclass()) {
           // root metaclass
           addMethod(cls, SEL_initialize, (IMP)&objc_noop_imp, "", NO);
       }
   
       // Attach categories.
       category_list *cats = unattachedCategoriesForClass(cls, true /*realizing*/);
       attachCategories(cls, cats, false /*don't flush caches*/);
   
       if (PrintConnecting) {
           if (cats) {
               for (uint32_t i = 0; i < cats->count; i++) {
                   _objc_inform("CLASS: attached category %c%s(%s)", 
                                isMeta ? '+' : '-', 
                                cls->nameForLogging(), cats->list[i].cat->name);
               }
           }
       }
       
       if (cats) free(cats);
   
   #if DEBUG
       // Debug: sanity-check all SELs; log method list contents
       for (const auto& meth : rw->methods) {
           if (PrintConnecting) {
               _objc_inform("METHOD %c[%s %s]", isMeta ? '+' : '-', 
                            cls->nameForLogging(), sel_getName(meth.name));
           }
           assert(sel_registerName(sel_getName(meth.name)) == meth.name); 
       }
   #endif
   }
   
   ```

7. 总结

   - 分类添加的方法可以"覆盖"原类方法
   - 同名 分类方法谁能生效取决于编译顺序
   - 名字相同的分类会引起编译报错

### 关联对象

1. 能否给分类添加 "成员变量"? 

2. 相关函数

   ``` c++
   id objc_getAssociatedObject(id object,const void *key);
   void objc_setAssociatedObject(id object,const void *key,id value,objc_AssociationPolicy policy);
   //移除对象的所有关联对象
   void objc_removeAssociatedObjects(id object);
   ```

3. "成员变量"被添加到哪里了

4. 关联对象的本质

   关联对象是由AssociationsManager 管理 并在AssociationsHashMap存储

   所有对象的关联内容都在同一个全局容器中

   |         ObjcAssociation         |
   | :-----------------------------: |
   | OBJC_ASSOCIATION_COPY_NONATOMIC |
   |            @"Hello"             |

   -->

   |             ObjectAssociationMap             |
   | :------------------------------------------: |
   | @selector(text)       \|     ObjcAssociation |

   -->

   |             AssociationsHashMap             |
   | :-----------------------------------------: |
   | DISGUISE(obj)     \|   ObjectAssociationMap |

5. 源码

   ```c++
   //runtime 750
   id objc_getAssociatedObject(id object, const void *key) {
       return _object_get_associative_reference(object, (void *)key);
   }
   
   
   void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) {
       _object_set_associative_reference(object, (void *)key, value, policy);
   }
   
   
   void objc_removeAssociatedObjects(id object) 
   {
       if (object && object->hasAssociatedObjects()) {
           _object_remove_assocations(object);
       }
   }
   
   id _object_get_associative_reference(id object, void *key) {
       id value = nil;
       uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
       {
           AssociationsManager manager;
           AssociationsHashMap &associations(manager.associations());
           disguised_ptr_t disguised_object = DISGUISE(object);
           AssociationsHashMap::iterator i = associations.find(disguised_object);
           if (i != associations.end()) {
               ObjectAssociationMap *refs = i->second;
               ObjectAssociationMap::iterator j = refs->find(key);
               if (j != refs->end()) {
                   ObjcAssociation &entry = j->second;
                   value = entry.value();
                   policy = entry.policy();
                   if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) {
                       objc_retain(value);
                   }
               }
           }
       }
       if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
           objc_autorelease(value);
       }
       return value;
   }
   
   void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
       // retain the new value (if any) outside the lock.
       ObjcAssociation old_association(0, nil);
       id new_value = value ? acquireValue(value, policy) : nil;
       {
           //关联对象管理类,C++实现的一个类
           AssociationsManager manager;
           //获取其维护的一个Hashmap,它是一个全局容器
           AssociationsHashMap &associations(manager.associations());
           disguised_ptr_t disguised_object = DISGUISE(object);//地址按位取反
           if (new_value) {
               // break any existing association.
               // 根据对象指针查找对应的一个ObjectAssociationMap结构的map
               //hashmap 通过 disguished_object 找到 map
               AssociationsHashMap::iterator i = associations.find(disguised_object);
               if (i != associations.end()) {
                   // secondary table exists 已经存在
                   ObjectAssociationMap *refs = i->second;
                   ObjectAssociationMap::iterator j = refs->find(key);
                   if (j != refs->end()) {
                       // key有值 要进行替换
                       old_association = j->second;
                       j->second = ObjcAssociation(policy, new_value);
                   } else {
                       (*refs)[key] = ObjcAssociation(policy, new_value);
                   }
               } else {
                   // create the new association (first time). 没找到得新建
                   ObjectAssociationMap *refs = new ObjectAssociationMap;
                   associations[disguised_object] = refs;
                   (*refs)[key] = ObjcAssociation(policy, new_value);
                   object->setHasAssociatedObjects();//设置object有了关联对象了
               }
           } else {
               // setting the association to nil breaks the association.
               // 传nil擦除 相应key的objcAssociation
               AssociationsHashMap::iterator i = associations.find(disguised_object);
               if (i !=  associations.end()) {
                   ObjectAssociationMap *refs = i->second;
                   ObjectAssociationMap::iterator j = refs->find(key);
                   if (j != refs->end()) {
                       old_association = j->second;
                       refs->erase(j);
                   }
               }
           }
       }
       // release the old value (outside of the lock).
       if (old_association.hasValue()) ReleaseValue()(old_association);
   }
   
   void _object_remove_assocations(id object) {
       vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
       {
           AssociationsManager manager;
           AssociationsHashMap &associations(manager.associations());
           if (associations.size() == 0) return;
           disguised_ptr_t disguised_object = DISGUISE(object);
           AssociationsHashMap::iterator i = associations.find(disguised_object);
           if (i != associations.end()) {
               // copy all of the associations that need to be removed.
               ObjectAssociationMap *refs = i->second;
               for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                   elements.push_back(j->second);
               }
               // remove the secondary table.
               delete refs;
               associations.erase(i);
           }
       }
       // the calls to releaseValue() happen outside of the lock.
       for_each(elements.begin(), elements.end(), ReleaseValue());
   }
   
   ```


### 扩展

1. 一般用扩展做什么 ?
   - 声明私有属性
   - 声明私有方法
   - 声明私有成员变量
2. 分类和扩展区别
   - 扩展是编译时决议 分类是运行时
   - 扩展 只以声明的形式存在,多数情况下寄生于宿主类的.m中. 分类可以有声明和实现
   - 不能为系统类添加扩展  但是可以为系统类添加分类

### 代理

1. 准确的说代理是一种软件设计模式

2. iOS当中以@protocol形式提现

3. 传递方式一对一

   ```sequence
   Title:代理(Delegate)
   participant 协议
   participant 代理方
   participant 委托方
   
   委托方->协议 : 要求代理方需要实现的接口
   委托方->代理方 : 调用代理方遵从的协议方法
   代理方->委托方 : 可能返回一个处理结果
   协议-> 代理方 : 按照协议实现方法
   
   
   
   
   
   ```

4. 一般声明为weak 以规避循环引用

### 通知

1. 是使用观察者模式来实现的用于跨层传递消息的机制

2. 传递方式一对多

3. 发送者->通知中心->传递给多个观察者

4. 如何实现通知机制?

   Notification_Map(notificationName,Observers_List)

   Observers_list(Observer,@selector()...)

5. 同步异步,通常是同步的回阻塞,可以通过notificationqueue 异步

### KVO

1. KVO是key-value observing的缩写

2. KVO是Objective-C对观察者设计模式的又一实现.

3. Apple使用了isa混写(isa-swizzling)来实现KVO. 创建子类 NSKVONotify_A 重写setter方法 isa指向

   ```objc
   -(void)setValue:(id)obj{
       [self willChangeValueForKey:@"keyPath"];
       [super setValue:obj];
       [self didChangeValueForKey:@"keyPath"];
   }
   ```

4. 通过kvc设置value能否生效  可以生效 最终会调用到setter方法

5. 通过成员变量直接赋值value能否生效 不能 必须调用willChangeValueForkey didChangeValueForkey

6. 成员变量直接修改需要手动添加KVO才会生效

### KVC

1. KVC是Key-value coding的缩写
   - -(id)valueForKey:(NSString*)key
   - -(void)setValue:(id)value forKey:(NSString *)key

2. 违背面向对象编程思想

3. valueForKey 流程

   ```flow
   St=>start: Start
   condi1=>condition: Accessor Method 
                         is exist?
   condi2=>condition: Instance var is exist? 
   +(BOOL)accessInstanceVariablesDirectly
   op1=>operation: Invoke
   op3=>operation: valueForUndefinedKey:
   op4=>operation: NSUndefinedKeyException
   end=>end: End
   St->condi1
   condi1(yes)->op1->end
   condi1(no)->condi2
   condi2(yes)->op1
   condi2(no)->op3->op4->end
   ```

4. Accessor Method

   - `<getKey>`
   - `<key>`
   - `<isKey>`

5. Instance var

   - _key
   - _isKey
   - isKey

6. setValueForKye

   ```flow
   st=>start: Start
   condi1=>condition: Setter Method is exist?
   condi2=>condition: Instance var is exist?
   +(BOOL)accessInstanceVariablesDirectly
   op1=>operation: Invoke
   op2=>operation: setValue:ForUndefinedKey:
   op3=>operation: NSUndefinedKeyException
   end=>end: End
   st->condi1
   condi1(yes)->op1->end
   condi1(no)->condi2
   condi2(yes)->op1
   condi2(no)->op2->op3->end
   ```


### 属性关键字

1. 读写权限

   - readonly
   - readwrite (default)

2. 原子性

   - atomic (default) 保证赋值和获取是线程安全的 比如对数组赋值就是安全的,但是对数组进行添加删除元素就不是
   - nonatomic

3. 引用计数

   - retain/strong
   - assign/unsafe_unretained
   - weak

4. assign

   - 修饰基本数据类型,如int,BOOL等
   - 修饰对象类型时,不改变其引用计数
   - 会产生悬垂指针

5. weak

   - 不改变被修饰对象的引用计数
   - 所指对象在释放后会自动置为nil

6. copy

   | 源码对象类型  |  拷贝方式   | 目标对象类型 | 拷贝类型(深/浅) |
   | :-----------: | :---------: | :----------: | :-------------: |
   |  mutable对象  |    copy     |    不可变    |     深拷贝      |
   |  mutable对象  | mutableCopy |     可变     |     深拷贝      |
   | immutable对象 |    copy     |    不可变    |     浅拷贝      |
   | immutable对象 | mutableCopy |     可变     |     深拷贝      |



### Objective-C 语言相关问题

1. MRC下如何重写retain修饰变量的setter方法?

   ``` objc
   @proterty (nonatomic,retain) id obj;
   - (void)setObj:(id)obj{
       if(_obj != obj){
           [_obj release];
           _obj= [obj retain];
       }
   }
   ```

2. 请简述分类实现原理

3. KVO的实现原理是怎样的?

4. 能否为分类添加成员变量?



## Runtime

1. 数据结构

   1. objc_object

      - `id = objc_object`
      - isa_t
      - 关于isa操作相关
      - 关联对象相关
      - 内存管理相关

   2. objc_class

      - Class = objc_class

      - 继承自objc_object

        ``` c++
        Class superClass;
        cache_t cache;
        class_data_bits_t bits;//变量属性方法在这里
        ```

   3. isa指针 

      - 共用体isa_t
      - 指针型isa isa的值代表Class的地址
      - 非指针型isa的 值的部分 代表Class的地址//部分为关键,寻址有关,非指针型一般在64位架构下存在,这时一些诸如nsnumber等所谓的小对象的值会存储到指针值里面.
      - isa指向
        - 关于对象,其指向类对象 实例--------->Class
        - 关于类对象,其指向元类对象 Class---------->metaClass

   4. cache_t

      - 用于快速查找方法执行函数
      - 是可增量扩展的哈希表结构
      - 是局部性原理的最佳应用 

   5. class_data_bits_t

      - class_data_bits_t 主要是对class_rw_t的封装

      - class_rw_t代表了类相关的读写信息,对class_ro_t的封装

      - class_ro_t 代表了类的只读信息

        ```objc
        struct class_rw_t {
            // Be warned that Symbolication knows the layout of this structure.
            uint32_t flags;
            uint32_t version;
        
            const class_ro_t *ro;
        
            method_array_t methods;
            property_array_t properties;
            protocol_array_t protocols;
        
            Class firstSubclass;
            Class nextSiblingClass;
        
            char *demangledName;
        
        #if SUPPORT_INDEXED_ISA
            uint32_t index;
        #endif
        
            void setFlags(uint32_t set) 
            {
                OSAtomicOr32Barrier(set, &flags);
            }
        
            void clearFlags(uint32_t clear) 
            {
                OSAtomicXor32Barrier(clear, &flags);
            }
        
            // set and clear must not overlap
            void changeFlags(uint32_t set, uint32_t clear) 
            {
                assert((set & clear) == 0);
        
                uint32_t oldf, newf;
                do {
                    oldf = flags;
                    newf = (oldf | set) & ~clear;
                } while (!OSAtomicCompareAndSwap32Barrier(oldf, newf, (volatile int32_t *)&flags));
            }
        };
        
        struct class_ro_t {
            uint32_t flags;
            uint32_t instanceStart;
            uint32_t instanceSize;
        #ifdef __LP64__
            uint32_t reserved;
        #endif
        
            const uint8_t * ivarLayout;
            
            const char * name;
            method_list_t * baseMethodList;
            protocol_list_t * baseProtocols;
            const ivar_list_t * ivars;
        
            const uint8_t * weakIvarLayout;
            property_list_t *baseProperties;
        
            method_list_t *baseMethods() const {
                return baseMethodList;
            }
        };
        ```

   6. method_t

       ``` c++
         struct method_t {
          SEL name;//方法名称
          const char *types;//函数返回值和参数组合 [返回值,参数1,参数2,参数3,...,参数n]
          MethodListIMP imp;//函数体
         
          struct SortBySELAddress :
              public std::binary_function<const method_t&,
                                          const method_t&, bool>
          {
              bool operator() (const method_t& lhs,
                               const method_t& rhs)
              { return lhs.name < rhs.name; }
          };
         };
       ```

   7. 数据结构总结

      ![Runtime数据结构](/Runtime数据结构.png)

2. 对象,类对象与原类对象(objc_class->objc_object)

   - 类对象存储实例方法列表信息

   - 元类对象存储方法列表等信息

   - ![image-20190124174604446](/image-20190124174604446-8323164.png)

   - 如果调用类方法没有对应的实现 但是NSObject有同名的实例方法的实现,会调用同名实例方法

   - 题目

     ```objc
     #import "Mobile.h"
     @interface Phone : Mobile
     @end
     @implementation Phone
     -(id) init
     {
       self = [super init];
         if(self){
             NSLog(@"%@",NSStringFromClass([self class]));
             NSLog(@"%@",NSStringFromClass([super class]));
         }
         return self;
     }
     @end
     ```

3. 消息传递

   1. `void objc_msgSend(void /*id self,SEL op,...*/)`

      `[self class] <------> objc_msgSend(self,@selector(class))`

   2. `void objc_msgSendSuper(void /*struct objc_super *super,SEL op,...*/)`

      ```c++
      struct objc_super {
          /// Specifies an instance of a class.
          __unsafe_unretained _Nonnull id receiver;
      
          /// Specifies the particular superclass of the instance to message. 
      #if !defined(__cplusplus)  &&  !__OBJC2__
          /* For compatibility with old objc-runtime.h header */
          __unsafe_unretained _Nonnull Class class;
      #else
          __unsafe_unretained _Nonnull Class super_class;
      #endif
          /* super_class is the first class to search */
      };
      
      [super class] <----> objc_msgSendSuper(super,@selector(class))
      ```

   3. 过程

      ![image-20190129162654739](/image-20190129162654739-8750414.png)

4. 方法缓存

   1. 给定值SEL,目标值是对应的bucket_t中的IMP.

   2. cache_key_t---f(key)--->bucket_t 哈希查找   f(key) = key & mask(举例哈希函数)

   3. 当前类中查找

      - 对于已排序好的列表,采用二分查找算法查找方法对应执行函数
      - 对于没有排序的列表,采用一般遍历查找方法对应执行函数

   4. 父类逐级查找

      ![image-20190129172117559](/image-20190129172117559.png)

5. 消息转发

   1. +resolveInstanceMethod: 

      ![image-20190130143821008](/image-20190130143821008-8830301.png)

6. Method-Swizzling

   1. 相关方法
      - class_getInstanceMethod
      - class_getClassMethod
      - method_exchangeImplementations

7. 动态添加方法

   1. performSelector:
   2. class_addMethod

8. 动态方法解析

   1. @dynamic
      - 动态运行时语言将函数决议推迟到运行时.
      - 编译时语言在编译期进行幻术决议

9. 题目

   1. [obj foo] 和 objc_msgSend()函数之间有什么关系?
   2. runtime 如何通过Selector找到对应的IMP地址的?
   3. 能否向编译后的类中增加实例变量? 不能,但是可以对动态添加的类中增加实例变量需要在objc_registerClassPair前.

## 内存管理

1. 内存布局

   - ![image-20190130153810710](/image-20190130153810710.png)

2. 内存管理方案

   1. 小对象(NSNumber等) taggedPointer

   2. NONPOINTER_ISA 64位架构下 isa本身占64比特位,本身不需要那么多,剩下的就存储内存管理相关的内容

      - arm64架构下

        ![image-20190130154457708](/image-20190130154457708-8834297.png)

        ![image-20190130154618071](/image-20190130154618071-8834378.png)

   3. 散列表(弱引用表,引用计数表)

      - Side Tables(结构)

        ![image-20190130154740143](/image-20190130154740143-8834460.png)

      - Side Table

        ![image-20190130154800513](/image-20190130154800513-8834480.png)

      - 为什么不是一个Side Tabel?

        存在效率问题

      - 分离锁 提高访问效率

      - 怎样实现款速分流?

        SideTables的本质是一张Hash表

        对象指针(Key)---(Hash函数)--->Side Table(Value)

3. 数据结构

   1. SpinLock_t

      - 是"忙等"的锁,当前锁如果被其它线程获取,当前线程会不断探测这个锁是否有被释放
      - 适用于轻量访问

   2. RefcountMap

      ptr-----DisguisedPtr(objc_object)---> size_t 哈希查找

   3. size_t

      ![image-20190130160137691](/image-20190130160137691-8835297.png)

   4. weak_table_t 也是hash表

      ![image-20190130160251794](/image-20190130160251794-8835371.png)

4. ARC & MRC

   1. MRC 手动引用计数
      - alloc
      - retain
      - release
      - retainCount
      - autorelease
      - dealloc
   2. ARC 自动引用计数
      - ARC是LLVM和Runtime协作的结果
      - ARC中禁止手动调用retain/release/retainCount/dealloc
      - ARC中新增weak,strong属性关键字

5. 引用计数

   1. alloc实现 

      经过一系列调用,最终调用了C函数calloc.此时并没有设置引用计数为1

   2. retain实现

      SideTabel& table = sideTables()[this];

      size_t& refcntStorage = table.refcnts[this];

      refcntStorage += SIDE_TABLE_RC_ONE;

   3. release 实现

      ```c++
      SideTable& table = SideTables()[this];
      RefCountMap::iterator it = table.refcnts.find(this);
      it->second -= SIDE_TABLE_RC_ONE;
      ```

   4. retainCount实现

      ```c++
      SideTable& table = SideTables()[this];
      size_t rrefcnt_result = 1;
      RefcountMap::iterator it = table.refcnts.fin(this);
      refcnt_result += it->second>>SIDE_TABLE_RC_SHIFT
      ```

   5. dealloc 实现

      1. 普通流程

         ![image-20190130165102672](/image-20190130165102672-8838262.png)

      2. object_dispose()流程

         ![image-20190130165215729](/image-20190130165215729-8838335.png)

      3. objc_destructInstance()

         ![image-20190130165356061](/image-20190130165356061-8838436.png)

      4. clearDeallocating()

         ![image-20190130165518293](/image-20190130165518293-8838518.png)

6. 弱引用

7. 自动释放池

8. 循环引用

## Blcok

## 多线程

## RunLoop

## 网络相关

## 设计模式

## 架构/框架

## 算法

## 第三方库

## 总结



 



