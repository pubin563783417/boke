<!-- README.md -->

# YYModel 源码解析

2019-01-02

[YYModel](https://github.com/ibireme/YYModel)

YYModel是一个序列化和反序列化库，说得更直白一点就是将NSString,NSData,NSDictionary形式的数据转换为具体类型的对象，以及将具体类型的对象转换为NSDictionary。

我们在没有开始看源码之前，我们猜测下如果这件事情要做应该怎么做？

* 将NSString,NSData,NSDictionary形式的数据转换为具体类型的对象
* 
首先我们需要将这些对象统一转换为NSDictionary，然后获取目标对象类的所有属性，创建一个目标对象，以目标对象的属性名为key到NSDictionary中去取值设置到目标对象上。最后返回这个对象。

* 将具体类型的对象转换为NSDictionary

这个其实是上一个过程的逆过程，首先需要获取当前对象类的信息，以及各个属性的getter/setter方法，使用getter方法取出当前对象当前属性的值，然后再将属性名为key，属性值为value存到NSDictionary中返回。

当然这仅仅是大体的构想，过程中还有很多细节需要考虑到，比如key和属性名的映射，容器类型属性的元素类型的指定方式等，我们接下来就来看下这部分代码，YYModel代码量不大，只要记住上面两点就可以把握大致的一个方向，如果对runtime机制比较熟悉会有更多的帮助，但是即使不知道也不妨碍你对整个代码的理解。

## 源码解析

## 1.JSON to Model

### 1.1 转换前准备工作 – 将JSON统一成NSDictionary

将JSON结构转换为Model是通过yy_modelWithJSON方法转换的，这里的json可以是NSString,NSData,NSDictionary

```c
+ (nullable instancetype)yy_modelWithJSON:(id)json;
```

yy_modelWithJSON作为总的入口会将这些类型统一通过_yy_dictionaryWithJSON转换为NSDictionary，再调用yy_modelWithDictionary将NSDictionary转换为Model返回。

```c
+ (instancetype)yy_modelWithJSON:(id)json {
    //将所有类型的数据都转换为NSDictionary类型
    NSDictionary *dic = [self _yy_dictionaryWithJSON:json];
    //将NSDictionary类型转换为model
    return [self yy_modelWithDictionary:dic];
}
```
```c
+ (NSDictionary *)_yy_dictionaryWithJSON:(id)json {
    if (!json || json == (id)kCFNull) return nil;
    NSDictionary *dic = nil;
    NSData *jsonData = nil;
    if ([json isKindOfClass:[NSDictionary class]]) {
        dic = json;
    } else if ([json isKindOfClass:[NSString class]]) {
        jsonData = [(NSString *)json dataUsingEncoding : NSUTF8StringEncoding];
    } else if ([json isKindOfClass:[NSData class]]) {
        jsonData = json;
    }
    //不论传入的json是NSDictionary，NSString还是NSData都将结果都转换为dic
    if (jsonData) {
        dic = [NSJSONSerialization JSONObjectWithData:jsonData options:kNilOptions error:NULL];
        if (![dic isKindOfClass:[NSDictionary class]]) dic = nil;
    }
    return dic;
}
```

_yy_dictionaryWithJSON 这块没什么好介绍的如果是NSDictionary就直接返回，如果是NSString或者NSData都通过JSONObjectWithData 转化为 NSDictionary对象，如果是其他类型就直接返回nil。

### 1.2 将NSDictionary 转换为Model对象

接下来的yy_modelWithDictionary开始就比较重要了，这里就开始对Model类进行分析，提取出它的全部属性，以及各种信息。

```c
+ (instancetype)yy_modelWithDictionary:(NSDictionary *)dictionary {
    
    if (!dictionary || dictionary == (id)kCFNull) return nil;
    if (![dictionary isKindOfClass:[NSDictionary class]]) return nil;
    
    Class cls = [self class];
    //获取当前类的属性
    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:cls];
    if (modelMeta->_hasCustomClassFromDictionary) {
        //根据dictionary来决定最后的cls
        cls = [cls modelCustomClassForDictionary:dictionary] ?: cls;
    }
    
    // 上面是获取当前类有哪些属性，哪些方法，以及实例方法，相当于建立了一个模版，
    // 就比方说我们定义了一个类，到上面为止，我们知道了它有a，b，c三个属性，每个属性的类型分别是NSString,NSInteger,BOOL 仅此而已，
    // 下面要做的就是创建出一个当前类的对象，并将传进来的字典里面的值赋给它的每个元素。这个在yy_modelSetWithDictionary中实现的。
    NSObject *one = [cls new];
    if ([one yy_modelSetWithDictionary:dictionary]) return one;
    return nil;
}
```

#### 1.2.1 提取Model信息

在上一个阶段将各种JSON格式转换为NSDictionary后，接下啦就从Model入手提取它的各种信息，我们看下_YYModelMeta的结构：

```c
@interface _YYModelMeta : NSObject {
    @package
    YYClassInfo *_classInfo;                //Model类的信息
    /// Key:mapped key and key path, Value:_YYModelPropertyMeta.
    NSDictionary *_mapper;                  //所有属性的key和keyPath
    /// Array<_YYModelPropertyMeta>, all property meta of this model.
    NSArray *_allPropertyMetas;            //所有属性的信息
    /// Array<_YYModelPropertyMeta>, property meta which is mapped to a key path.
    NSArray *_keyPathPropertyMetas;        //属于keyPath的属性列表
    /// Array<_YYModelPropertyMeta>, property meta which is mapped to multi keys.
    NSArray *_multiKeysPropertyMetas;     //一个属性多个key的属性列表
    /// The number of mapped key (and key path), same to _mapper.count.
    NSUInteger _keyMappedCount;           //全部属性数量
    /// Model class type.
    YYEncodingNSType _nsType;             //Model类的对象
    
    //这个类实现覆盖方法的情况
    BOOL _hasCustomWillTransformFromDictionary;
    BOOL _hasCustomTransformFromDictionary;
    BOOL _hasCustomTransformToDictionary;
    BOOL _hasCustomClassFromDictionary;
}
@end
```

从_YYModelMeta结构中我们可以看到_YYModelMeta既有key与属性的映射关系，又有各个属性的Meta信息，所以基本上就可以通过明确知道字典中的每个key对应的value值对应的属性。只要有这个关系就可以将NSDictionary中的值映射到Model属性上。我们接下来看下这部分源码：

```c
+ (instancetype)metaWithClass:(Class)cls {
    if (!cls) return nil;
    static CFMutableDictionaryRef cache;
    static dispatch_once_t onceToken;
    static dispatch_semaphore_t lock;
    //建立缓存
    dispatch_once(&onceToken, ^{
        cache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        lock = dispatch_semaphore_create(1);
    });
    //通过cls从缓存中取出meta
    dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
    _YYModelMeta *meta = CFDictionaryGetValue(cache, (__bridge const void *)(cls));
    dispatch_semaphore_signal(lock);
    //如果之前没有缓存过或者缓存的信息需要更新
    if (!meta || meta->_classInfo.needUpdate) {
        //通过cls 创建出一个 _YYModelMeta
        meta = [[_YYModelMeta alloc] initWithClass:cls];
        if (meta) {
            //将新建的_YYModelMeta添加到缓存供下次使用
            dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
            CFDictionarySetValue(cache, (__bridge const void *)(cls), (__bridge const void *)(meta));
            dispatch_semaphore_signal(lock);
        }
    }
    return meta;
}
```

在第一调用metaWithClass的时候会创建一个缓存，并且为这个缓存添加一个dispatch_semaphor信号量，为了避免每次都进行查询，这里会将每次查询的数据缓存到新建的缓存中，以加快查询速度。

```c
- (instancetype)initWithClass:(Class)cls {
    
    //用于存储类的信息 包括当前类，父类，当前属性，实例变量，方法
    YYClassInfo *classInfo = [YYClassInfo classInfoWithClass:cls];
    if (!classInfo) return nil;
    self = [super init];
    
    // 获取属性黑名单
    NSSet *blacklist = nil;
    if ([cls respondsToSelector:@selector(modelPropertyBlacklist)]) {
        NSArray *properties = [(id<YYModel>)cls modelPropertyBlacklist];
        if (properties) {
            blacklist = [NSSet setWithArray:properties];
        }
    }
    
    // 获取属性白名单
    NSSet *whitelist = nil;
    if ([cls respondsToSelector:@selector(modelPropertyWhitelist)]) {
        NSArray *properties = [(id<YYModel>)cls modelPropertyWhitelist];
        if (properties) {
            whitelist = [NSSet setWithArray:properties];
        }
    }
    
    // 获取容器的元素对象，存储到genericMapper key为属性名 value为该容器类里面元素的类型
    NSDictionary *genericMapper = nil;
    if ([cls respondsToSelector:@selector(modelContainerPropertyGenericClass)]) {
        genericMapper = [(id<YYModel>)cls modelContainerPropertyGenericClass];
        if (genericMapper) {
            NSMutableDictionary *tmp = [NSMutableDictionary new];
            [genericMapper enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
                if (![key isKindOfClass:[NSString class]]) return;
                Class meta = object_getClass(obj);
                if (!meta) return;
                if (class_isMetaClass(meta)) {
                    //key为属性名 value为该容器类里面元素的类型
                    tmp[key] = obj;
                } else if ([obj isKindOfClass:[NSString class]]) {
                    Class cls = NSClassFromString(obj);
                    if (cls) {
                        tmp[key] = cls;
                    }
                }
            }];
            genericMapper = tmp;
        }
    }
    

    NSMutableDictionary *allPropertyMetas = [NSMutableDictionary new];
    YYClassInfo *curClassInfo = classInfo;
    //遍历当前类及其父类的属性，
    while (curClassInfo && curClassInfo.superCls != nil) { // recursive parse super class, but ignore root class (NSObject/NSProxy)
        for (YYClassPropertyInfo *propertyInfo in curClassInfo.propertyInfos.allValues) {
            if (!propertyInfo.name) continue;
            //只有不在黑名单并且再白名单中的属性才会被加到allPropertyMetas
            if (blacklist && [blacklist containsObject:propertyInfo.name]) continue;
            if (whitelist && ![whitelist containsObject:propertyInfo.name]) continue;
            _YYModelPropertyMeta *meta = [_YYModelPropertyMeta metaWithClassInfo:classInfo
                                                                    propertyInfo:propertyInfo
                                                                         generic:genericMapper[propertyInfo.name]];
            //meta必须有值，并且有getter/setters
            if (!meta || !meta->_name) continue;
            if (!meta->_getter || !meta->_setter) continue;
            //已经存在allPropertyMetas中了则不再继续
            if (allPropertyMetas[meta->_name]) continue;
            //将meta存到allPropertyMetas中
            allPropertyMetas[meta->_name] = meta;
        }
        curClassInfo = curClassInfo.superClassInfo;
    }
    if (allPropertyMetas.count) _allPropertyMetas = allPropertyMetas.allValues.copy;
    //将所有的类属性都添加到allPropertyMetas
    
    
    
    // 建立映射关系
    NSMutableDictionary *mapper = [NSMutableDictionary new];
    NSMutableArray *keyPathPropertyMetas = [NSMutableArray new];
    NSMutableArray *multiKeysPropertyMetas = [NSMutableArray new];
    
    //处理映射
    if ([cls respondsToSelector:@selector(modelCustomPropertyMapper)]) {
        //获得自定义映射字典，它用于解决json文件中关键字和定义的类的属性不一致的问题。
        /*
         + (NSDictionary *) modelCustomPropertyMapper {
            return @{@"errnoTest"类属性 : @"errno"json中的字段};
        }*/
        NSDictionary *customMapper = [(id <YYModel>)cls modelCustomPropertyMapper];
        [customMapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyName, NSString *mappedToKey, BOOL *stop) {
            //取出原来key对应的属性信息
            _YYModelPropertyMeta *propertyMeta = allPropertyMetas[propertyName];
            if (!propertyMeta) return;
            //从allPropertyMetas移除
            [allPropertyMetas removeObjectForKey:propertyName];
            //要被映射到的属性名字
            if ([mappedToKey isKindOfClass:[NSString class]]) {
                if (mappedToKey.length == 0) return;
                //只有一个key
                propertyMeta->_mappedToKey = mappedToKey;
                
                //去掉空keyPath
                NSArray *keyPath = [mappedToKey componentsSeparatedByString:@"."];
                for (NSString *onePath in keyPath) {
                    if (onePath.length == 0) {
                        NSMutableArray *tmp = keyPath.mutableCopy;
                        [tmp removeObject:@""];
                        keyPath = tmp;
                        break;
                    }
                }
                
                if (keyPath.count > 1) {
                    //有多个path的key，例如xxx.xxxx.xxx
                    propertyMeta->_mappedToKeyPath = keyPath;
                    [keyPathPropertyMetas addObject:propertyMeta];
                }
                propertyMeta->_next = mapper[mappedToKey] ?: nil;
                mapper[mappedToKey] = propertyMeta;
                
            } else if ([mappedToKey isKindOfClass:[NSArray class]]) {
                
                NSMutableArray *mappedToKeyArray = [NSMutableArray new];
                for (NSString *oneKey in ((NSArray *)mappedToKey)) {
                    if (![oneKey isKindOfClass:[NSString class]]) continue;
                    if (oneKey.length == 0) continue;
                    
                    NSArray *keyPath = [oneKey componentsSeparatedByString:@"."];
                    if (keyPath.count > 1) {
                        [mappedToKeyArray addObject:keyPath];
                    } else {
                        [mappedToKeyArray addObject:oneKey];
                    }
                    
                    if (!propertyMeta->_mappedToKey) {
                        propertyMeta->_mappedToKey = oneKey;
                        propertyMeta->_mappedToKeyPath = keyPath.count > 1 ? keyPath : nil;
                    }
                }
                if (!propertyMeta->_mappedToKey) return;
                
                propertyMeta->_mappedToKeyArray = mappedToKeyArray;
                [multiKeysPropertyMetas addObject:propertyMeta];
                
                propertyMeta->_next = mapper[mappedToKey] ?: nil;
                mapper[mappedToKey] = propertyMeta;
            }
        }];
    }
    
    //这些是上面还没处理的，处理后添加到mapper，所以mapper包含了全部的属性
    [allPropertyMetas enumerateKeysAndObjectsUsingBlock:^(NSString *name, _YYModelPropertyMeta *propertyMeta, BOOL *stop) {
        propertyMeta->_mappedToKey = name;
        //如果有其他值绑定在同一个key上则将它赋给next
        propertyMeta->_next = mapper[name] ?: nil;
        //将属性添加到mapper
        mapper[name] = propertyMeta;
    }];
    
    //将上述的属性信息添加到_mapper以及_keyPathPropertyMetas，_multiKeysPropertyMetas
    if (mapper.count) _mapper = mapper;
    if (keyPathPropertyMetas) _keyPathPropertyMetas = keyPathPropertyMetas;
    if (multiKeysPropertyMetas) _multiKeysPropertyMetas = multiKeysPropertyMetas;
    
    //赋值到对应的属性上
    _classInfo = classInfo;
    _keyMappedCount = _allPropertyMetas.count;
    _nsType = YYClassGetNSType(cls);
    _hasCustomWillTransformFromDictionary = ([cls instancesRespondToSelector:@selector(modelCustomWillTransformFromDictionary:)]);
    _hasCustomTransformFromDictionary = ([cls instancesRespondToSelector:@selector(modelCustomTransformFromDictionary:)]);
    _hasCustomTransformToDictionary = ([cls instancesRespondToSelector:@selector(modelCustomTransformToDictionary:)]);
    _hasCustomClassFromDictionary = ([cls respondsToSelector:@selector(modelCustomClassForDictionary:)]);
    
    return self;
}
```

嗯 是的 上面是_YYModelMeta的构造方法，哈哈，长吧。在_YYModelMeta的构造方法中主要是负责填充前面介绍过的_YYModelMeta中的属性。我们接下来将它拆成一段一段进行介绍：

* 获取当前类的属性信息

```c
//用于存储类的信息 包括当前类，父类，当前属性，实例变量，方法
YYClassInfo *classInfo = [YYClassInfo classInfoWithClass:cls];
if (!classInfo) return nil;
```

如果看过runtime源码的同学应该对接下来的这部分代码理解会比较透彻，如果不属性也无防在这里先了解下，后面看runtime的代码的时候会恍然大悟，看源码其实很多时候第一遍可能理解的只是一个大概，不要太去纠结细节，后面在某项技术补上后，或者在项目上遇到类似的问题的时候会有更深的理解，所以看不懂埋着头看下去，一遍不行，再来一遍，就是这种傻办法慢慢积累。

```c
@interface YYClassInfo : NSObject
@property (nonatomic, assign, readonly) Class cls; ///< //当前YYClassInfo 所对应的cls
@property (nullable, nonatomic, assign, readonly) Class superCls; ///< 当前类的父类
@property (nullable, nonatomic, assign, readonly) Class metaCls;  ///< 当前类的meta对象
@property (nonatomic, readonly) BOOL isMeta; ///< 当前类是否是meta类
@property (nonatomic, strong, readonly) NSString *name; ///< 类名
@property (nullable, nonatomic, strong, readonly) YYClassInfo *superClassInfo; ///< 父类信息
@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassIvarInfo *> *ivarInfos; ///< 实例变量信息
@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassMethodInfo *> *methodInfos; ///< 方法信息
@property (nullable, nonatomic, strong, readonly) NSDictionary<NSString *, YYClassPropertyInfo *> *propertyInfos; ///< 属性信息

//.......
@end
```

接下来看下classInfoWithClass：

```c
+ (instancetype)classInfoWithClass:(Class)cls {
    if (!cls) return nil;
    //........
    
    //创建缓存
    dispatch_once(&onceToken, ^{
        classCache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        metaCache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        lock = dispatch_semaphore_create(1);
    });
    //获取class信息
    dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
    YYClassInfo *info = CFDictionaryGetValue(class_isMetaClass(cls) ? metaCache : classCache, (__bridge const void *)(cls));
    //查看是否需要更新，如果需要更新的话则调用info的_update方法
    if (info && info->_needUpdate) {
        [info _update];
    }
    dispatch_semaphore_signal(lock);
    //如果缓存没有则新建后添加到缓存中
    if (!info) {
        info = [[YYClassInfo alloc] initWithClass:cls];
        if (info) {
            dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
            CFDictionarySetValue(info.isMeta ? metaCache : classCache, (__bridge const void *)(cls), (__bridge const void *)(info));
            dispatch_semaphore_signal(lock);
        }
    }
    return info;
}
```

嗯，熟悉吧，还是使用了缓存，很明显数据的序列化其实是一个特别频繁的任务，基本上每次网络请求回来后都需要经过序列化后进行交付，如果每次都进行类信息的提取将会是很影响性能的任务。

```c
- (instancetype)initWithClass:(Class)cls {
    if (!cls) return nil;
    self = [super init];
    _cls = cls;
    _superCls = class_getSuperclass(cls);
    _isMeta = class_isMetaClass(cls);
    if (!_isMeta) {
        _metaCls = objc_getMetaClass(class_getName(cls));
    }
    _name = NSStringFromClass(cls);
    [self _update];
    _superClassInfo = [self.class classInfoWithClass:_superCls];
    return self;
}

- (void)_update {
    //.....
    
    Class cls = self.cls;
    unsigned int methodCount = 0;
    //获取当前类的方法数
    Method *methods = class_copyMethodList(cls, &methodCount);
    if (methods) {
        NSMutableDictionary *methodInfos = [NSMutableDictionary new];
        _methodInfos = methodInfos;
        for (unsigned int i = 0; i < methodCount; i++) {
            YYClassMethodInfo *info = [[YYClassMethodInfo alloc] initWithMethod:methods[i]];
            //key --> 方法名  value 方法信息
            if (info.name) methodInfos[info.name] = info;
        }
        free(methods);
    }
    //获取当前的属性
    unsigned int propertyCount = 0;
    objc_property_t *properties = class_copyPropertyList(cls, &propertyCount);
    if (properties) {
        NSMutableDictionary *propertyInfos = [NSMutableDictionary new];
        _propertyInfos = propertyInfos;
        for (unsigned int i = 0; i < propertyCount; i++) {
            YYClassPropertyInfo *info = [[YYClassPropertyInfo alloc] initWithProperty:properties[i]];
            //key --> 属性名  value 属性信息
            if (info.name) propertyInfos[info.name] = info;
        }
        free(properties);
    }
    //获取当前对象的实例变量
    unsigned int ivarCount = 0;
    Ivar *ivars = class_copyIvarList(cls, &ivarCount);
    if (ivars) {
        NSMutableDictionary *ivarInfos = [NSMutableDictionary new];
        _ivarInfos = ivarInfos;
        for (unsigned int i = 0; i < ivarCount; i++) {
            YYClassIvarInfo *info = [[YYClassIvarInfo alloc] initWithIvar:ivars[i]];
            if (info.name) ivarInfos[info.name] = info;
        }
        free(ivars);
    }
    
    //......
    //更新结束
    _needUpdate = NO;
}
```

上面一整段代码就是负责使用runtime方法提取class中的属性信息，实例遍历信息，方法信息，父类信息等，

* 通过属性黑白名单对属性进行过滤

```c
// 获取属性黑名单
NSSet *blacklist = nil;
if ([cls respondsToSelector:@selector(modelPropertyBlacklist)]) {
    NSArray *properties = [(id<YYModel>)cls modelPropertyBlacklist];
    if (properties) {
        blacklist = [NSSet setWithArray:properties];
    }
}

// 获取属性白名单
NSSet *whitelist = nil;
if ([cls respondsToSelector:@selector(modelPropertyWhitelist)]) {
    NSArray *properties = [(id<YYModel>)cls modelPropertyWhitelist];
    if (properties) {
        whitelist = [NSSet setWithArray:properties];
    }
}

// 获取容器的元素对象，存储到genericMapper key为属性名 value为该容器类里面元素的类型
NSDictionary *genericMapper = nil;
if ([cls respondsToSelector:@selector(modelContainerPropertyGenericClass)]) {
    genericMapper = [(id<YYModel>)cls modelContainerPropertyGenericClass];
    if (genericMapper) {
        NSMutableDictionary *tmp = [NSMutableDictionary new];
        [genericMapper enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
            if (![key isKindOfClass:[NSString class]]) return;
            Class meta = object_getClass(obj);
            if (!meta) return;
            if (class_isMetaClass(meta)) {
                //key为属性名 value为该容器类里面元素的类型
                tmp[key] = obj;
            } else if ([obj isKindOfClass:[NSString class]]) {
                Class cls = NSClassFromString(obj);
                if (cls) {
                    tmp[key] = cls;
                }
            }
        }];
        genericMapper = tmp;
    }
}
    
NSMutableDictionary *allPropertyMetas = [NSMutableDictionary new];
YYClassInfo *curClassInfo = classInfo;
//遍历当前类及其父类的属性，
while (curClassInfo && curClassInfo.superCls != nil) { // recursive parse super class, but ignore root class (NSObject/NSProxy)
    for (YYClassPropertyInfo *propertyInfo in curClassInfo.propertyInfos.allValues) {
        if (!propertyInfo.name) continue;
        //只有不在黑名单并且再白名单中的属性才会被加到allPropertyMetas
        if (blacklist && [blacklist containsObject:propertyInfo.name]) continue;
        if (whitelist && ![whitelist containsObject:propertyInfo.name]) continue;
        _YYModelPropertyMeta *meta = [_YYModelPropertyMeta metaWithClassInfo:classInfo
                                                                propertyInfo:propertyInfo
                                                                        generic:genericMapper[propertyInfo.name]];
        //meta必须有值，并且有getter/setters
        if (!meta || !meta->_name) continue;
        if (!meta->_getter || !meta->_setter) continue;
        //已经存在allPropertyMetas中了则不再继续
        if (allPropertyMetas[meta->_name]) continue;
        //将meta存到allPropertyMetas中
        allPropertyMetas[meta->_name] = meta;
    }
    curClassInfo = curClassInfo.superClassInfo;
}
if (allPropertyMetas.count) _allPropertyMetas = allPropertyMetas.allValues.copy;
//将所有的类属性都添加到allPropertyMetas
```

上面代码虽然长但是完成的工作却很简单就是从类中提取出黑白名单，然后对当前类的全部属性，包括父对象在内的全部属性进行过滤，对于不在白名单，在黑名单的属性进行过滤。当中穿插了一个容器类元素对象类型的获取过程，如果我们的某个属性是容器对象：比如数组，元素类型这部分其实信息是丢失的。需要我们通过外部方式指定，YYmodel中会通过覆写modelContainerPropertyGenericClass方法，指定某个属性的元素类型。最终存放在genericMapper，在初始化_YYModelPropertyMeta的时候就可以通过属性最为key，查到对应的generic。添加到_YYModelProperty中。这里还必须注意每个属性都必须具备getter/Setter 方法。最终将各个属性添加到_allPropertyMetas上。

#### 1.2.2 提取Model信息

```c
// 建立映射关系
NSMutableDictionary *mapper = [NSMutableDictionary new];
NSMutableArray *keyPathPropertyMetas = [NSMutableArray new];
NSMutableArray *multiKeysPropertyMetas = [NSMutableArray new];

//处理映射
if ([cls respondsToSelector:@selector(modelCustomPropertyMapper)]) {
    //获得自定义映射字典，它用于解决json文件中关键字和定义的类的属性不一致的问题。
    NSDictionary *customMapper = [(id <YYModel>)cls modelCustomPropertyMapper];
    [customMapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyName, NSString *mappedToKey, BOOL *stop) {
        //取出原来key对应的属性信息
        _YYModelPropertyMeta *propertyMeta = allPropertyMetas[propertyName];
        if (!propertyMeta) return;
        //从allPropertyMetas移除
        [allPropertyMetas removeObjectForKey:propertyName];
        if ([mappedToKey isKindOfClass:[NSString class]]) {
            if (mappedToKey.length == 0) return;
            // 1.最简单的形式
            propertyMeta->_mappedToKey = mappedToKey;
            //去掉空keyPath
            NSArray *keyPath = [mappedToKey componentsSeparatedByString:@"."];
            for (NSString *onePath in keyPath) {
                if (onePath.length == 0) {
                    NSMutableArray *tmp = keyPath.mutableCopy;
                    [tmp removeObject:@""];
                    keyPath = tmp;
                    break;
                }
            }
            if (keyPath.count > 1) {
                //2. 有多个path的key，例如xxx.xxxx.xxx
                propertyMeta->_mappedToKeyPath = keyPath;
                //添加到keyPathPropertyMetas
                [keyPathPropertyMetas addObject:propertyMeta];
            }
            propertyMeta->_next = mapper[mappedToKey] ?: nil;
            mapper[mappedToKey] = propertyMeta;
            
        } else if ([mappedToKey isKindOfClass:[NSArray class]]) {
            //3.映射的对象是数组类型的时候
            NSMutableArray *mappedToKeyArray = [NSMutableArray new];
            for (NSString *oneKey in ((NSArray *)mappedToKey)) {
                if (![oneKey isKindOfClass:[NSString class]]) continue;
                if (oneKey.length == 0) continue;
                
                NSArray *keyPath = [oneKey componentsSeparatedByString:@"."];
                if (keyPath.count > 1) {
                    [mappedToKeyArray addObject:keyPath];
                } else {
                    [mappedToKeyArray addObject:oneKey];
                }
                
                if (!propertyMeta->_mappedToKey) {
                    propertyMeta->_mappedToKey = oneKey;
                    propertyMeta->_mappedToKeyPath = keyPath.count > 1 ? keyPath : nil;
                }
            }
            if (!propertyMeta->_mappedToKey) return;
            //添加到multiKeysPropertyMetas
            propertyMeta->_mappedToKeyArray = mappedToKeyArray;
            [multiKeysPropertyMetas addObject:propertyMeta];
            
            propertyMeta->_next = mapper[mappedToKey] ?: nil;
            mapper[mappedToKey] = propertyMeta;
        }
    }];
}
```

我们结合下面的例子来解析上面的代码：

```c
{
    "n":"Harry Pottery",
    "ext" : {
        "desc" : "A book written by J.K.Rowing."
    },
    "ID" : 100010
}

+ (NSDictionary *)modelCustomPropertyMapper {
    return @{@"name" : @"n",
             @"desc" : @"ext.desc",
             @"bookID" : @[@"id",@"ID",@"book_id"]};
}
```

上面的例子三个属性分别对应代码中标出的1，2，3 三种例子，我们分别看这三种情况：

**情况一 最简单的映射:**

这种没啥好介绍的就是将modelCustomPropertyMapper对应的value存到propertyMeta->_mappedToKey

```c
// 1.最简单的形式
propertyMeta->_mappedToKey = mappedToKey;
```

**情况二 keyPath的映射:**

```c
NSArray *keyPath = [mappedToKey componentsSeparatedByString:@"."];
for (NSString *onePath in keyPath) {
    if (onePath.length == 0) {
        NSMutableArray *tmp = keyPath.mutableCopy;
        [tmp removeObject:@""];
        keyPath = tmp;
        break;
    }
}
if (keyPath.count > 1) {
    //2. 有多个path的key，例如xxx.xxxx.xxx
    propertyMeta->_mappedToKeyPath = keyPath;
    //添加到keyPathPropertyMetas
    [keyPathPropertyMetas addObject:propertyMeta];
}
propertyMeta->_next = mapper[mappedToKey] ?: nil;
mapper[mappedToKey] = propertyMeta;
```

这里和上面的区别是它会将类似ext.desc这种形式的路径转化为数组@[@”ext”,@”desc”],然后存到_mappedToKeyPath中并添加到keyPathPropertyMetas，以后如果要找到这种带路径的属性可以直接从keyPathPropertyMetas找。

**情况三 多映射类型key的映射:**

```c
//3.映射的对象是数组类型的时候
NSMutableArray *mappedToKeyArray = [NSMutableArray new];
for (NSString *oneKey in ((NSArray *)mappedToKey)) {
    //........
    NSArray *keyPath = [oneKey componentsSeparatedByString:@"."];
    if (keyPath.count > 1) {
        [mappedToKeyArray addObject:keyPath];
    } else {
        [mappedToKeyArray addObject:oneKey];
    }
    //先填充_mappedToKey和_mappedToKeyPath
    if (!propertyMeta->_mappedToKey) {
        propertyMeta->_mappedToKey = oneKey;
        propertyMeta->_mappedToKeyPath = keyPath.count > 1 ? keyPath : nil;
    }
}
if (!propertyMeta->_mappedToKey) return;
//添加到multiKeysPropertyMetas
propertyMeta->_mappedToKeyArray = mappedToKeyArray;
[multiKeysPropertyMetas addObject:propertyMeta];

propertyMeta->_next = mapper[mappedToKey] ?: nil;
mapper[mappedToKey] = propertyMeta;
```

我们拿 @”bookID” : @[@”id”,@”ID”,@”book_id”]作为例子，经过上面处理后propertyMeta->_mappedToKey = @”ID”,propertyMeta->_mappedToKeyPath = nil, propertyMeta->_mappedToKeyArray = @[@”id”,@”ID”,@”book_id”] 最后propertyMeta会被添加到multiKeysPropertyMetas数组中。

**情况四 其余不在modelCustomPropertyMapper中指定的映射:**

```c
[allPropertyMetas enumerateKeysAndObjectsUsingBlock:^(NSString *name, _YYModelPropertyMeta *propertyMeta, BOOL *stop) {
    propertyMeta->_mappedToKey = name;
    //如果有其他值绑定在同一个key上则将它赋给next
    propertyMeta->_next = mapper[name] ?: nil;
    //将属性添加到mapper
    mapper[name] = propertyMeta;
}];
```

上面三种情况都是处理在modelCustomPropertyMapper中特别指定的属性，而上面的这种情况是没有特殊指定的属性，这种情况下就是将属性名赋给_mappedToKey。

综上几种情况，所有的映射关系都存在于_mappedToKey，_mappedToKeyPath，_mappedToKeyArray这三个属性中,比如下面这种情况：

```c
// Model:
@interface Book : NSObject
@property NSString *name;
@property NSString *test;
@property NSString *desc;
@property NSString *bookID;
@end
@implementation Book
+ (NSDictionary *)modelCustomPropertyMapper {
    return @{@"name" : @"n",
             @"desc" : @"ext.desc",
             @"bookID" : @[@"id",@"ID",@"book_id"]};
}
@end
```

属性name的_mappedToKey = @”n”,_mappedToKeyPath = nil,_mappedToKeyArray = nil
属性test的_mappedToKey = @”test”,_mappedToKeyPath = nil,_mappedToKeyArray = nil
属性desc的_mappedToKey = @”ext.desc”,_mappedToKeyPath = @[@”ext”,@”desc”],_mappedToKeyArray = nil
属性bookID的_mappedToKey = @”id”,_mappedToKeyPath = nil ,_mappedToKeyArray = @[@”id”,@”ID”,@”book_id”]

#### 1.2.3 使用NSDictionary的数据填充Model

我们前面已经获取到了Model类的class结构信息，并且完成了属性黑白名单的过滤，以及属性名和JSON中字段名的对应关系，接下来我们就可以使用Model 类创建出一个Model,并从JSON (NSDictionary)中取出对应的值，对Model对象进行填充，最后再将生成的model对象返回就完成了整个序列化过程,这部分代码位于yy_modelSetWithDictionary：

```c
- (BOOL)yy_modelSetWithDictionary:(NSDictionary *)dic {
    //.....
    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:object_getClass(self)];
    //.....
    //构建context 上下文中包括modelMeta model的各种映射信息，model 要填充的model对象， dictionary 包含数据的字典
    ModelSetContext context = {0};
    context.modelMeta = (__bridge void *)(modelMeta);
    context.model = (__bridge void *)(self);
    context.dictionary = (__bridge void *)(dic);
    //开始将dictionary数据填充到model上，这里最关键的就是ModelSetWithPropertyMetaArrayFunction方法。
    if (modelMeta->_keyMappedCount >= CFDictionaryGetCount((CFDictionaryRef)dic)) {

        CFDictionaryApplyFunction((CFDictionaryRef)dic, ModelSetWithDictionaryFunction, &context);

        if (modelMeta->_keyPathPropertyMetas) {
            CFArrayApplyFunction((CFArrayRef)modelMeta->_keyPathPropertyMetas,
                                 CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_keyPathPropertyMetas)),
                                 ModelSetWithPropertyMetaArrayFunction,
                                 &context);
        }

        if (modelMeta->_multiKeysPropertyMetas) {
            CFArrayApplyFunction((CFArrayRef)modelMeta->_multiKeysPropertyMetas,
                                 CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_multiKeysPropertyMetas)),
                                 ModelSetWithPropertyMetaArrayFunction,
                                 &context);
        }
    } else {
        CFArrayApplyFunction((CFArrayRef)modelMeta->_allPropertyMetas,
                             CFRangeMake(0, modelMeta->_keyMappedCount),
                             ModelSetWithPropertyMetaArrayFunction,
                             &context);
    }
    //........
    return YES;
}
```

我们先来看下：

```c
CFDictionaryApplyFunction((CFDictionaryRef)dic, ModelSetWithDictionaryFunction, &context);
```

它实际上是对dic，也就是当前JSON所对应的NSDictionary的所有元素，应用ModelSetWithDictionaryFunction方法,并在每次调用中将context传递进去，下面是ModelSetWithDictionaryFunction的定义：CFDictionaryApplyFunction 会将NSDictionary中的每个元素的key，作为ModelSetWithDictionaryFunction第一个参数，value作为第二个参数，最后是一个context用户存对应的公共数据。比如这里的modelMeta，model，dictionary，ModelSetWithDictionaryFunction就是将dictionary各个元素取出来，使用key的映射关系来找到modelMeta中的属性meta，取出dictionary中的值通过属性meta中的setter将值设置到model中。我们来看下这部分代码：

```c
static void ModelSetWithDictionaryFunction(const void *_key, const void *_value, void *_context) {
    ModelSetContext *context = _context;
    __unsafe_unretained _YYModelMeta *meta = (__bridge _YYModelMeta *)(context->modelMeta);
    //取出单个属性
    __unsafe_unretained _YYModelPropertyMeta *propertyMeta = [meta->_mapper objectForKey:(__bridge id)(_key)];
    //取出model
    __unsafe_unretained id model = (__bridge id)(context->model);
    while (propertyMeta) {
        if (propertyMeta->_setter) {
            //将从dictionary中获取到的某个值赋给某个model对象的某个属性
            ModelSetValueForProperty(model/*对象*/, (__bridge __unsafe_unretained id)_value/*某个属性的值*/, propertyMeta/*属性信息*/);
        }
        propertyMeta = propertyMeta->_next;
    };
}
```

对于_keyPathPropertyMetas 以及 _multiKeysPropertyMetas通过调用ModelSetWithPropertyMetaArrayFunction来设置的，最终也是归到ModelSetValueForProperty

```c
static void ModelSetWithPropertyMetaArrayFunction(const void *_propertyMeta, void *_context) {
    ModelSetContext *context = _context;
    __unsafe_unretained NSDictionary *dictionary = (__bridge NSDictionary *)(context->dictionary);
    __unsafe_unretained _YYModelPropertyMeta *propertyMeta = (__bridge _YYModelPropertyMeta *)(_propertyMeta);
    if (!propertyMeta->_setter) return;
    id value = nil;
    
    //通过属性信息里面的key的映射关系拿到字典里面对应的value值。
    if (propertyMeta->_mappedToKeyArray) {
        value = YYValueForMultiKeys(dictionary, propertyMeta->_mappedToKeyArray);
    } else if (propertyMeta->_mappedToKeyPath) {
        value = YYValueForKeyPath(dictionary, propertyMeta->_mappedToKeyPath);
    } else {
        value = [dictionary objectForKey:propertyMeta->_mappedToKey];
    }
    //将value赋给model中的属性
    if (value) {
        __unsafe_unretained id model = (__bridge id)(context->model);
        ModelSetValueForProperty(model, value, propertyMeta);
    }
}
```

ModelSetValueForProperty是整个库中算得上是比较长的代码了，之所以长是因为类型比较多？我们取其中简单的一种作为例子—- JSON中的某个值为字符类型，Model中的某个属性为NSString,或者NSMutableString类型，这时候通过objc_msgSend 完 model中调用对应属性的setter方法将值设置到model上。

```c
static void ModelSetValueForProperty(__unsafe_unretained id model,
                                     __unsafe_unretained id value,
                                     __unsafe_unretained _YYModelPropertyMeta *meta) {
    
    //获取meta的的类型，也就是要将字典里面的值，转成的目标类型。这里为什么用objc_msgSend，是因为可能有些属性会自定义不同的setter。
    if (meta->_isCNumber) {
        //.....
    } else if (meta->_nsType) {
        if (value == (id)kCFNull) {
            //......
        } else {
            switch (meta->_nsType) {
                case YYEncodingTypeNSString:
                case YYEncodingTypeNSMutableString: {
                    if ([value isKindOfClass:[NSString class]]) {
                        if (meta->_nsType == YYEncodingTypeNSString) {
                            ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, value);
                        } else {
                            ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, ((NSString *)value).mutableCopy);
                        }
                    } 
                //......
                default: break;
                }
            }
        }
    }
}
```

## 2. Model to JSON
在yy_modelToJSONObject方法的开头有下面一段注释：

> Apple said:
The top level object is an NSArray or NSDictionary.
All objects are instances of NSString, NSNumber, NSArray, NSDictionary, or NSNull.
All dictionary keys are instances of NSString.
Numbers are not NaN or infinity.

翻译成中文就是，标准的JSON顶层是一个NSArray或者NSDictionary，并且所有的对象都是NSString，NSNumber，NSArray，NSDictionary，NSNull这些类的实例，并且字典中的key都是NSString类型的实例，数值都是非NaN或者无穷，这其实大家都比较清楚这里强调下对后续代码的理解会有一定的帮助，我们继续看下怎么将Model转换为NSDictionary Model，入口是yy_modelToJSONObject

```c
- (id)yy_modelToJSONObject {
    id jsonObject = ModelToJSONObjectRecursive(self);
    if ([jsonObject isKindOfClass:[NSArray class]]) return jsonObject;
    if ([jsonObject isKindOfClass:[NSDictionary class]]) return jsonObject;
    return nil;
}
```

在yy_modelToJSONObject方法中就先调用ModelToJSONObjectRecursive来递归地将当前对象转换为jsonObject，只有顶层对象是NSArray和NSDictionary才返回，否则都是nil。因此最关键的转换应该在ModelToJSONObjectRecursive方法中。

```c
static id ModelToJSONObjectRecursive(NSObject *model) {
    // 对于简单的NSString，NSNumber，nill类型
    if (!model || model == (id)kCFNull) return model;
    if ([model isKindOfClass:[NSString class]]) return model;
    if ([model isKindOfClass:[NSNumber class]]) return model;
    if ([model isKindOfClass:[NSURL class]]) return ((NSURL *)model).absoluteString;
    if ([model isKindOfClass:[NSAttributedString class]]) return ((NSAttributedString *)model).string;
    if ([model isKindOfClass:[NSDate class]]) return [YYISODateFormatter() stringFromDate:(id)model];
    if ([model isKindOfClass:[NSData class]]) return nil;
    
    //如果当前对象是NSDictionary
    if ([model isKindOfClass:[NSDictionary class]]) {
        // 简单的字典
        if ([NSJSONSerialization isValidJSONObject:model]) return model;
        // 对象是复杂的对象
        NSMutableDictionary *newDic = [NSMutableDictionary new];
        [((NSDictionary *)model) enumerateKeysAndObjectsUsingBlock:^(NSString *key, id obj, BOOL *stop) {
            NSString *stringKey = [key isKindOfClass:[NSString class]] ? key : key.description;
            if (!stringKey) return;
            //复杂对象需要递归转换
            id jsonObj = ModelToJSONObjectRecursive(obj);
            if (!jsonObj) jsonObj = (id)kCFNull;
            newDic[stringKey] = jsonObj;
        }];
        return newDic;
    }
    //如果当前对象是NSSet
    if ([model isKindOfClass:[NSSet class]]) {
        // 简单的NSSet
        NSArray *array = ((NSSet *)model).allObjects;
        if ([NSJSONSerialization isValidJSONObject:array]) return array;
        
        // 对象是复杂的对象
        NSMutableArray *newArray = [NSMutableArray new];
        for (id obj in array) {
            if ([obj isKindOfClass:[NSString class]] || [obj isKindOfClass:[NSNumber class]]) {
                [newArray addObject:obj];
            } else {
                id jsonObj = ModelToJSONObjectRecursive(obj);
                if (jsonObj && jsonObj != (id)kCFNull) [newArray addObject:jsonObj];
            }
        }
        return newArray;
    }
    //如果当前对象是NSArray
    if ([model isKindOfClass:[NSArray class]]) {
        // 简单的NSArray
        if ([NSJSONSerialization isValidJSONObject:model]) return model;
        
        // 对象是复杂的对象
        NSMutableArray *newArray = [NSMutableArray new];
        for (id obj in (NSArray *)model) {
            if ([obj isKindOfClass:[NSString class]] || [obj isKindOfClass:[NSNumber class]]) {
                [newArray addObject:obj];
            } else {
                id jsonObj = ModelToJSONObjectRecursive(obj);
                if (jsonObj && jsonObj != (id)kCFNull) [newArray addObject:jsonObj];
            }
        }
        return newArray;
    }
    
    //获取Model信息
    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:[model class]];
    if (!modelMeta || modelMeta->_keyMappedCount == 0) return nil;
    
    NSMutableDictionary *result = [[NSMutableDictionary alloc] initWithCapacity:64];
    __unsafe_unretained NSMutableDictionary *dic = result; // avoid retain and release in block
    [modelMeta->_mapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyMappedKey, _YYModelPropertyMeta *propertyMeta, BOOL *stop) {
        if (!propertyMeta->_getter) return;
        // 通过属性信息中的_getter方法使用objc_msgSend从model中获取当前对应属性的值，值获取到了，mapkey也知道了就可以构造返回的dictionary了。
        id value = nil;
        if (propertyMeta->_isCNumber) {
            value = ModelCreateNumberFromProperty(model, propertyMeta);
        } else if (propertyMeta->_nsType) {
            id v = ((id (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
            value = ModelToJSONObjectRecursive(v);
        } else {
            switch (propertyMeta->_type & YYEncodingTypeMask) {
                case YYEncodingTypeObject: {
                    id v = ((id (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                    value = ModelToJSONObjectRecursive(v);
                    if (value == (id)kCFNull) value = nil;
                } break;
                case YYEncodingTypeClass: {
                    Class v = ((Class (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                    value = v ? NSStringFromClass(v) : nil;
                } break;
                case YYEncodingTypeSEL: {
                    SEL v = ((SEL (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                    value = v ? NSStringFromSelector(v) : nil;
                } break;
                default: break;
            }
        }
        if (!value) return;
        
        //根据是否有_mappedToKeyPath将上面获取到的值加入到字典中。
        if (propertyMeta->_mappedToKeyPath) {
            NSMutableDictionary *superDic = dic;
            NSMutableDictionary *subDic = nil;
            for (NSUInteger i = 0, max = propertyMeta->_mappedToKeyPath.count; i < max; i++) {
                NSString *key = propertyMeta->_mappedToKeyPath[i];
                if (i + 1 == max) { // end
                    if (!superDic[key]) superDic[key] = value;
                    break;
                }
                subDic = superDic[key];
                if (subDic) {
                    if ([subDic isKindOfClass:[NSDictionary class]]) {
                        subDic = subDic.mutableCopy;
                        superDic[key] = subDic;
                    } else {
                        break;
                    }
                } else {
                    subDic = [NSMutableDictionary new];
                    superDic[key] = subDic;
                }
                superDic = subDic;
                subDic = nil;
            }
        } else {
            if (!dic[propertyMeta->_mappedToKey]) {
                dic[propertyMeta->_mappedToKey] = value;
            }
        }
    }];
    if (modelMeta->_hasCustomTransformToDictionary) {
        BOOL suc = [((id<YYModel>)model) modelCustomTransformToDictionary:dic];
        if (!suc) return nil;
    }
    return result;
}
```

ModelToJSONObjectRecurs代码很长我们分段进行解析：

**简单类型**

```c
 if (!model || model == (id)kCFNull) return model;
if ([model isKindOfClass:[NSString class]]) return model;
if ([model isKindOfClass:[NSNumber class]]) return model;
if ([model isKindOfClass:[NSURL class]]) return ((NSURL *)model).absoluteString;
if ([model isKindOfClass:[NSAttributedString class]]) return ((NSAttributedString *)model).string;
if ([model isKindOfClass:[NSDate class]]) return [YYISODateFormatter() stringFromDate:(id)model];
if ([model isKindOfClass:[NSData class]]) return nil;
```

这部分就不介绍了。

**NSDictionary, NSSet,NSArray 容器类型**

```c
if ([model isKindOfClass:[NSDictionary class]]) {
        // 简单的字典
        if ([NSJSONSerialization isValidJSONObject:model]) return model;
        // 对象是复杂的对象
        NSMutableDictionary *newDic = [NSMutableDictionary new];
        [((NSDictionary *)model) enumerateKeysAndObjectsUsingBlock:^(NSString *key, id obj, BOOL *stop) {
            NSString *stringKey = [key isKindOfClass:[NSString class]] ? key : key.description;
            if (!stringKey) return;
            //复杂对象需要递归转换
            id jsonObj = ModelToJSONObjectRecursive(obj);
            if (!jsonObj) jsonObj = (id)kCFNull;
            newDic[stringKey] = jsonObj;
        }];
        return newDic;
    }
    //如果当前对象是NSSet
    if ([model isKindOfClass:[NSSet class]]) {
        //.....
    }
    //如果当前对象是NSArray
    if ([model isKindOfClass:[NSArray class]]) {
        //.....
    }
 ```
    
对于NSDictionary, NSSet,NSArray 容器类型处理的过程是类似的，我们这里以NSDictionary为例子进行分析，如果model是NSDictionary会遍历每个元素，对每个值递归调用ModelToJSONObjectRecursive后存储到newDic中作为value。最后返回newDic。

**其他对象类型**

对于我们自定义的类型我们处理逻辑如下：首先我们会先通过metaWithClass进行抽取Model的详细信息，然后对model全部属性进行遍历，通过调用每个属性的getter方法提取出属性值，并通过属性名与NSDictionary key的映射关系确定key后将值存到key对应的value中，从而完成整个过程。

```c
_YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:[model class]];

NSMutableDictionary *result = [[NSMutableDictionary alloc] initWithCapacity:64];
__unsafe_unretained NSMutableDictionary *dic = result; // avoid retain and release in block
[modelMeta->_mapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyMappedKey, _YYModelPropertyMeta *propertyMeta, BOOL *stop) {
    if (!propertyMeta->_getter) return;
    // 通过属性信息中的_getter方法使用objc_msgSend从model中获取当前对应属性的值，值获取到了，mapkey也知道了就可以构造返回的dictionary了。
    id value = nil;
    if (propertyMeta->_isCNumber) {
        value = ModelCreateNumberFromProperty(model, propertyMeta);
    } else if (propertyMeta->_nsType) {
        id v = ((id (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
        value = ModelToJSONObjectRecursive(v);
    } else {
        switch (propertyMeta->_type & YYEncodingTypeMask) {
            case YYEncodingTypeObject: {
                id v = ((id (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                value = ModelToJSONObjectRecursive(v);
                if (value == (id)kCFNull) value = nil;
            } break;
            case YYEncodingTypeClass: {
                Class v = ((Class (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                value = v ? NSStringFromClass(v) : nil;
            } break;
            case YYEncodingTypeSEL: {
                SEL v = ((SEL (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                value = v ? NSStringFromSelector(v) : nil;
            } break;
            default: break;
        }
    }
    if (!value) return;

    //根据是否有_mappedToKeyPath将上面获取到的值加入到字典中。
    if (propertyMeta->_mappedToKeyPath) {
        NSMutableDictionary *superDic = dic;
        NSMutableDictionary *subDic = nil;
        for (NSUInteger i = 0, max = propertyMeta->_mappedToKeyPath.count; i < max; i++) {
            NSString *key = propertyMeta->_mappedToKeyPath[i];
            if (i + 1 == max) { // end
                if (!superDic[key]) superDic[key] = value;
                break;
            }
            subDic = superDic[key];
            if (subDic) {
                if ([subDic isKindOfClass:[NSDictionary class]]) {
                    subDic = subDic.mutableCopy;
                    superDic[key] = subDic;
                } else {
                    break;
                }
            } else {
                subDic = [NSMutableDictionary new];
                superDic[key] = subDic;
            }
            superDic = subDic;
            subDic = nil;
        }
    } else {
        if (!dic[propertyMeta->_mappedToKey]) {
            dic[propertyMeta->_mappedToKey] = value;
        }
    }
}
```

## 3. 其他JSON 转 Model 方法

除了上面介绍的JSON 转 Model 方法，适用于如下类型：

```json
{
    "key1":"value1",
    "key2":"value2",
    "key3":"value3",
    "key4":"value4",
    "key5":"value5",
}
```


当然YYModel还适合序列化对象数组类型的JSON比如：

```json
[
    {
        "key1":"value1",
        "key2":"value2",
    },
    {
        "key3":"value3",
        "key4":"value4",
    },
    {
        "key5":"value5",
        "key6":"value6",
    }
]
```

这里仅仅贴出代码，有了上面的介绍理解下面的代码应该不会有任何困难。

```c
+ (NSArray *)yy_modelArrayWithClass:(Class)cls json:(id)json {
    if (!json) return nil;
    NSArray *arr = nil;
    NSData *jsonData = nil;
    if ([json isKindOfClass:[NSArray class]]) {
        arr = json;
    } else if ([json isKindOfClass:[NSString class]]) {
        jsonData = [(NSString *)json dataUsingEncoding : NSUTF8StringEncoding];
    } else if ([json isKindOfClass:[NSData class]]) {
        jsonData = json;
    }
    if (jsonData) {
        arr = [NSJSONSerialization JSONObjectWithData:jsonData options:kNilOptions error:NULL];
        if (![arr isKindOfClass:[NSArray class]]) arr = nil;
    }
    return [self yy_modelArrayWithClass:cls array:arr];
}

+ (NSArray *)yy_modelArrayWithClass:(Class)cls array:(NSArray *)arr {
    if (!cls || !arr) return nil;
    NSMutableArray *result = [NSMutableArray new];
    for (NSDictionary *dic in arr) {
        if (![dic isKindOfClass:[NSDictionary class]]) continue;
        NSObject *obj = [cls yy_modelWithDictionary:dic];
        if (obj) [result addObject:obj];
    }
    return result;
}
```

还可以解析下面类型的JSON,

```json
{
    "obj1":
    {
        "key1":"value1",
        "key2":"value2",
    },
    "obj2":
    {
        "key3":"value3",
        "key4":"value4",
    },
    "obj3":
    {
        "key5":"value5",
        "key6":"value6",
    }
}
```

代码如下和上面的那个其实是类似的，只不过是顶层元素换成字典罢了，这类个人在项目中用得不多。

```c
+ (NSDictionary *)yy_modelDictionaryWithClass:(Class)cls json:(id)json {
    if (!json) return nil;
    NSDictionary *dic = nil;
    NSData *jsonData = nil;
    if ([json isKindOfClass:[NSDictionary class]]) {
        dic = json;
    } else if ([json isKindOfClass:[NSString class]]) {
        jsonData = [(NSString *)json dataUsingEncoding : NSUTF8StringEncoding];
    } else if ([json isKindOfClass:[NSData class]]) {
        jsonData = json;
    }
    if (jsonData) {
        dic = [NSJSONSerialization JSONObjectWithData:jsonData options:kNilOptions error:NULL];
        if (![dic isKindOfClass:[NSDictionary class]]) dic = nil;
    }
    return [self yy_modelDictionaryWithClass:cls dictionary:dic];
}

+ (NSDictionary *)yy_modelDictionaryWithClass:(Class)cls dictionary:(NSDictionary *)dic {
    if (!cls || !dic) return nil;
    NSMutableDictionary *result = [NSMutableDictionary new];
    for (NSString *key in dic.allKeys) {
        if (![key isKindOfClass:[NSString class]]) continue;
        NSObject *obj = [cls yy_modelWithDictionary:dic[key]];
        if (obj) result[key] = obj;
    }
    return result;
}
```

除了上面介绍的方法外，YYModel还提供了极大减轻个人工作量的方法：

```c
// 深度拷贝
- (id)yy_modelCopy;
// NSCoding
- (void)yy_modelEncodeWithCoder:(NSCoder *)aCoder;
- (id)yy_modelInitWithCoder:(NSCoder *)aDecoder;
// Hash值生成
- (NSUInteger)yy_modelHash;
// 判断是否一致
- (BOOL)yy_modelIsEqual:(id)model;
// 当前对象的描述
- (NSString *)yy_modelDescription;
```


## 4. 总结

YYModel代码就解析到这里，其实大体的思路就是，解析Model对象信息，明确NSDictionary key 与 Model属性名之间的映射关系，通过这种映射关系，新建一个model对象，使用model对象的setter方法将NSDictionary中的值设置到model中，从而完成JSON转Model 的过程。Model 转 JSON是通过model的getter方法，取出对应属性的值，然后通过NSDictionary key和model属性名的映射关系，确定对应的key，然后将model值设置到NSDictionary中。


