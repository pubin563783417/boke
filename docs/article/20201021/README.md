<!-- README.md -->

# Swift 面向协议编程

2020-10-21

> 在WWDC15上，苹果宣布Swift是世界上第一门面向协议编程(POP)语言。相比与传统的面向对象编程 (OOP)，POP 显得更加灵活。RxSwift、ReactorKit 核心也是面向协议编程的。


## 一、什么是POP

要弄清楚什么是面向协议(POP),我们应该先知道什么是Swift协议？
我们定义一个简单的Swift协议如下：

```swift
protocol Runable {
    var name: String { get }
    func run()
}
```

代码中定义名为Runable协议,包含一个name属性,以及一个run方法的定义
所谓协议，就是一组属性和/或方法的定义，而如果某个具体类型想要遵守一个协议，那它需要实现这个协议所定义的所有这些内容。协议实际上做的事情不过是"关于实现的约定"。

面向协议编程(POP)是Swift2.0引入的一种新的编程范式。POP就是通过协议扩展，协议继承和协议组合的方式来设计需要编写的代码。

## 二、为什么要使用面向协议编程(POP)

Swift是一门面向对象的语言,类已经满足我们所有的需求，功能也十分强大。为什么还要使用POP?

首先在Swift中，值类型优先于类。然而，面向对象的概念不能很好地与结构体和枚举一起工作: 因为结构体和枚举不能够被继承。因此，作为面向对象的一大特征—继承就不能够应用于值类型了。

再者,实际开发工程中,我们经常会遇到如下场景:
假设我们有一个ViewController，它继承自UIViewController，我们向其新添加一个方法 customMethod：

```swift
class ViewController: UIViewController {
    //新添加
    func customMethod() {
    }
}
```

这个时候我们有另外一个继承自UITableViewController的OtherViewController,同样也需要向其添加方法customMethod

```swift
class OtherViewController: UITableViewController {
    //新添加
    func customMethod() {
    }
}
```

这里就存在一个问题:很难在不同继承关系的类里共用代码。
我们的关注点customMethod位于两条继承链 UIViewController -> ViewCotroller 和 UIViewController -> UITableViewController -> AnotherViewController 的横切面上。面向对象是一种不错的抽象方式，但是肯定不是最好的方式。它无法描述两个不同事物具有某个相同特性这一点。在这里，特性的组合要比继承更贴切事物的本质。

这种情况,我们有如下几种方法解决:


- Copy & Paste
这是一个糟糕的解决方案,我们应该尽量避免这种做法。


- BaseViewController
在一个继承自 UIViewController 的 BaseViewController 上添加需要共享的代码,或者在 UIViewController 上添加 extension。这是目前很多人常用的解决方法,但是如果不断这么做，会让所谓的BaseViewController 很快变成垃圾堆。职责不明确，任何东西都能扔进 Base，你完全不知道哪些类走了 Base，而这个“超级类”对代码的影响也会不可预估。


- 依赖注入
通过外界传入一个带有 customMethod的对象，用新的类型来提供这个功能。这是一个稍好的方式，但是引入额外的依赖关系，可能也是我们不太愿意看到的。


- 多继承
当然，Swift 是不支持多继承的。不过如果有多继承的话，我们确实可以从多个父类进行继承，并将customMethod添加到合适的地方,但这又会带来其他问题。


总的来说,面向协议编程(POP) 带来的好处如下:

- 结构体、枚举等值类型也可以使用
- 以继承多个协议，弥补 swift 中类单继承的不足
- 增强代码的可扩展性，减少代码的冗余
- 让项目更加组件化，代码可读性更高
- 让无需的功能代码组成一个功能块，更便于单元测试。

## 三、使用POP解决上述问题

定义一个含有customMethod的协议ex:

```swift
protocol ex {
    func customMethod();
}
```

复制代码在实际类型遵守这个协议:

```swift
extension ViewController :ex {
    func customMethod() {
        //
    }
}


extension OtherViewController : ex {
    func customMethod() {
        //
    }
}
```

复制代码这样的实现和Copy & Paste的方式一样,必须要在ViewController和OtherViewController都写一遍。有什么方法可以解决这问题呢?那就是协议扩展

可以在 extension ex 中为 customMethod 添加一个实现:

```swift
extension ex {
    func customMethod() {
        //
    }
}
```

有这个协议扩展,只需要声明ViewController和OtherViewController遵循ex,就可以直接使用customMethod了。

## 四、协议的特性及使用

### 协议扩展：

- 1.提供协议方法的默认实现和协议属性的默认值，从而使它们成为可选；符合协议的类型可以提供自己的实现，也可以使用默认的实现。
- 2.添加协议中未声明的附加方法实现，并且实现协议的任何类型都可以使用到这些附加方法。这样就可以给遵循协议的类型添加特定的方法

```swift
protocol Entity {
    var name: String {get set}
    static func uid() -> String
}

extension Entity {
    static func uid() -> String {
        return UUID().uuidString
    }
}

struct Order: Entity {
    var name: String
    let uid: String = Order.uid()
}
let order = Order(name: "My Order")
print(order.uid)
```

### 协议继承

协议可以从其他协议继承，然后在它继承的需求之上添加功能，因此可以提供更细粒度和更灵活的设计。

```swift
protocol Persistable: Entity {
    func write(instance: Entity, to filePath: String)
    init?(by uid: String)
}

struct InMemoryEntity: Entity {
    var name: String
}

struct PersistableEntity: Persistable {
    var name: String
    func write(instance: Entity, to filePath: String) { // ...
    }  
    init?(by uid: String) {
        // try to load from the filesystem based on id
    }
}
```

### 协议的组合

类、结构体和枚举可以符合多个协议，它们可以采用多个协议的默认实现。这在概念上类似于多继承。这种组合的方式不仅比将所有需要的功能压缩到一个基类中更灵活，而且也适用于值类型。

```swift
struct MyEntity: Entity, Equatable, CustomStringConvertible {
    var name: String
    // Equatable
    public static func ==(lhs: MyEntity, rhs: MyEntity) -> Bool {
        return lhs.name == rhs.name
    }
    // CustomStringConvertible
    public var description: String {
        return "MyEntity: \(name)"
    }
}
let entity1 = MyEntity(name: "42")
print(entity1)
let entity2 = MyEntity(name: "42")
assert(entity1 == entity2, "Entities shall be equal")
```



