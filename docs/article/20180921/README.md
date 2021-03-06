<!-- README.md -->

# iOS @property属性详解

2018-09-21

**property 声明了ivar（实例变量）和存取方法（access method ＝ getter + setter)。所有关键词都是用于声明或者控制这些。其主要的作用就在于封装对象中的数据。 Objective-C 对象通常会把其所需要的数据保存为各种实例变量。实例变量一般通过getter来访问，setter用于写入变量值。**

- 声明一个变量：

 ```c
 @property(nonatomic,strong) NSString *url;
 ```
 
等于声明了一个 _url实例变量，一个getUrl方法和一个setUrl方法。

## Object-c属性关键词

|  1  | 2  | 3 | 4 | 5 |
|  ---- | ---- | ---- | ---- | ---- |
| nonatomic | atomic | readonly | writeonly | readwrite |
| assign | retain |  copy | strong | weak |
| unsafe_unretained | nonnull |  nullable | null_resettable | synthesize |
| dynamic |
	
		
## nonatomic、atomic关键词

- atomic 是默认关键字。表示该属性是线程同步的，这个属性是为了保证程序在多线程情况下，编译器会自动生成一些互斥加锁代码，避免该变量的读写不同步问题。如果不是特殊情况不要用这个关键词，会影响性能。

- nonatomic 非线程同步。这回让编译器少生成一些互斥加锁代码，可以提高效率。

## readonly、writeonly、readwrite关键词

- 用于属性的读写控制。在写Framework的时候我们不希望用户更改属性值，就可以通过这个来控制。

- readwrite 默认关键字，表示可读可写，可以调用get 和 set 方法。

- readonly 只读属性。表示只可以调用get 方法。

- writeonly 只写属性。表示这可以调用set方法。

## strong

- 强引用，对象的引用计数+1.ARC环境为默认属性类型
- 只要引用计数为0 的时候对象才会被释放。

## weak

- 弱引用，不会改变引用计数。当对象释放的时候，指针会变为nil。
- 一般会在代理和block里面使用weak，以防止循环引用。
- 当一个对象已经被强引用过之后，另一个对象要指向这个对象时可以使用weak修饰。比如：自定义 IBOutlet控件属性一般也使用weak，当然也可以使用strong
## assign

- 非对象类型(值类型)一般使用此关键字.不会改变引用计数。
- 如果用assign修饰一个对象，当对象释放的时候，指针的地址还是存在的，并没有被置为nil，就会造成野指针，如果再去使用这个对象程序就会崩溃了。

 ```c
  @property (nonatomic, assign) CGFload value
 ```
  
## copy

- 拷贝一个新的对象，新对象的引用计数+1，原对象不变
- 一般用于修饰一个block或者需要拷贝对象

```c
  @property (nonatomic, copy) void (^block ) (void)

  @property (nonatomic, copy) NSObject *Object
```

## retain

对象的引用计数+1。ARC下用strong代替。

## unsafe_unretained

- 不会自动设置为nil，如果对象被释放了，再进行访问，程序会crash.不会影响对象的引用计数
- 它和assign是等价的

## __block与__weak的区别

- __block是用来修饰一个变量，这个变量就可以在block中被修改

- __block：使用 __block修饰的变量在block代码块中会被retain（ARC下会retain，MRC下不会retain）

- __weak：使用__weak修饰的变量不会在block代码块中被retain

- Object-c 下block的使用方式：

```c
__weak typeof(self) weakSelf = self;
block = ^(void) {
    __strong __typeof(weakSelf)strongSelf = weakSelf;
    [strongSelf action];
};
```


## nonnull、nullable、null_resettable 关键词

- 属性声明关键词，与Swift中的?和!类似。主要是在oc 和 swift混编的时候，用于修饰oc对象。如果为nullable，swift会把对象认为是可选值。
- nullable 会被swift任务可选值
- nonnull 会被swift任务非可选值。
- null_resettable 表示setter是nullable，可选的，但是getter是nonnull，一定会有值返回
- 这些知识声明，程序运行的时候还有有可能向nonnull 赋值的。所以混编的时候把对象都修饰为nullable比较安全

如果修饰变量名需要用__nullable或者__nonnull

```c
- (void)completeBlock:(nullable void (^)(NSError * __nullable error))block;
```

## @synthesize和@dynamic

- synthesize的表示如果没有手动实现setter方法和getter方法，那么编译器会自动为你加上这两个方法。
- dynamic告诉编译器：属性的setter与getter方法由用户自己实现，不自动生成。


