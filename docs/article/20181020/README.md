<!-- README.md -->

# Association 关联对象

2018-10-20

## 1. 关联对象

### 1.1 使用场景
默认情况下，由于分类底层结构的限制，不能直接给 Category 添加成员变量，但是可以通过关联对象间接实现 Category 有成员变量的效果。

[传送门：OC - Category 和 Extension](https://www.jianshu.com/p/1eca0353ceab)

### 1.2 使用方法
```c
#import "Person.h"
@interface Person (Test)
@property (nonatomic, assign) int height;
@end

#import "Person+Test.h"
#import <objc/runtime.h>
@implementation Person (Test)
- (void)setHeight:(int)height
{
    objc_setAssociatedObject(self, @selector(height), [NSNumber numberWithInt:height], OBJC_ASSOCIATION_ASSIGN);
}
- (int)height
{
    return [objc_getAssociatedObject(self, @selector(height)) intValue];
}
@end
```

### 1.2.1 相关 API

**objc_setAssociatedObject**

使用给定的key和关联策略为给定的对象设置关联的value。

> void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);

**objc_getAssociatedObjec **

返回给定key的给定对象关联的value。

> id objc_getAssociatedObject(id object, const void *key);



**objc_removeAssociatedObjects**

删除给定对象的所有关联。

> void objc_removeAssociatedObjects(id object);

如果只想移除给定对象的某个key的关联，可以使用objc_setAssociatedObject给参数value传值nil。

>objc_setAssociatedObject(self, @selector(height), nil, OBJC_ASSOCIATION_ASSIGN);

### 1.2.2 objc_AssociationPolicy 关联策略
```c
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```

|  objc_AssociationPolicy   | 对应的属性修饰符  |
|  ----  | ----  |
| OBJC_ASSOCIATION_ASSIGN  | assign |
| OBJC_ASSOCIATION_RETAIN_NONATOMIC  | strong, nonatomic |
| OBJC_ASSOCIATION_COPY_NONATOMIC  | copy, nonatomic |
| OBJC_ASSOCIATION_RETAIN  | strong, atomic |
| OBJC_ASSOCIATION_COPY  | copy, atomic |


### 1.2.3 key 的常见用法

```c
// ①
static const void *MyKey = &MyKey;
objc_setAssociatedObject(object, MyKey, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
objc_getAssociatedObject(object, MyKey];

// ② 
static const char MyKey;
objc_setAssociatedObject(object, &MyKey, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
objc_getAssociatedObject(object, &MyKey];

// ③ 使用属性名作为 key
#define MYHEIGHT @"height"
objc_setAssociatedObject(object, MYHEIGHT, value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
objc_getAssociatedObject(object, MYHEIGHT];

// ④ 使用 getter 方法的 SEL 作为 key（可读性高，有智能提示）
objc_setAssociatedObject(object, @selector(getter), value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
objc_getAssociatedObject(object, @selector(getter)];
// 或使用隐式参数 _cmd
objc_getAssociatedObject(object, _cmd];
```


## 2. 关联对象的原理

在Runtime源码objc4中，有关关联对象的代码都在文件objc-references.mm中。

**实现关联对象技术的核心对象**

* AssociationsManager
* AssociationsHashMap
* ObjectAssociationMap
* ObjcAssociation

```c
class AssociationsManager {
    static AssociationsHashMap *_map;
};
class AssociationsHashMap : public unordered_map<disguised_ptr_t, ObjectAssociationMap>
class ObjectAssociationMap : public std::map<void *, ObjcAssociation>
class ObjcAssociation {
    uintptr_t _policy;
    id _value;
};
```


**数据结构**

**AssociationsManager**

* 关联对象并不是存储在关联对象本身内存中，而是存储在全局统一的一个容器中；
* 由 AssociationsManager 管理并在它维护的一个单例 Hash 表 AssociationsHashMap 中存储；
* 使用 AssociationsManagerLock 自旋锁保证了线程安全。


**AssociationsHashMap**

* 一个单例的 Hash 表，存储 disguised_ptr_t 和 ObjectAssociationMap 之间的映射。
* disguised_ptr_t 是根据 object 生成，但不存在引用关系。

> disguised_ptr_t disguised_object = DISGUISE(object);

**ObjectAssociationMap**

* 存储 key 和 ObjcAssociation 之间的映射。


**ObjcAssociation

* 存储着关联策略 policy 和关联对象的值 value。


**objc_setAssociatedObject**

* ① 实例化一个  AssociationsManager 对象，它维护了一个单例 Hash 表 AssociationsHashMap 对象；
* ② 实例化一个 AssociationsHashMap 对象，它维护 disguised_ptr_t 和 ObjectAssociationMap 对象之间的关系；
* ③ 根据object生成一个 disguised_ptr_t 对象；
* ④ 根据 disguised_ptr_t 获取对应的 ObjectAssociationMap 对象，它存储key和 ObjcAssociation 之间的映射；
* ⑤ 根据policy和value创建一个 ObjcAssociation 对象，并存储在 ObjectAssociationMap 中；
* ⑥ 如果传进来的value为 nil，则在 ObjectAssociationMap 中删除该 ObjcAssociation 对象。



objc_getAssociatedObject

① 实例化一个  AssociationsManager 对象；
② 实例化一个 AssociationsHashMap 对象；
③ 根据object生成一个 disguised_ptr_t 对象；
④ 根据 disguised_ptr_t 获取对应的 ObjectAssociationMap 对象；
⑤ 根据 key 获取到它所对应的 ObjcAssociation 对象；
⑥ 返回 ObjcAssociation 中的 value。


**objc_removeAssociatedObjects**

* ① 实例化一个  AssociationsManager 对象；
* ② 实例化一个 AssociationsHashMap 对象；
* ③ 根据object生成一个 disguised_ptr_t 对象；
* ④ 根据 disguised_ptr_t 获取对应的 ObjectAssociationMap 对象；
* ⑤ 删除该 ObjectAssociationMap 对象。


**acquireValue**

根据policy来对value进行retain或者copy操作。






