<!-- README.md -->

<!-- # 目录索引 -->

# 目录

<div class="mulu">
     <a class="a-item"  href="#/article/20220224/README.md">
        <h2>  Rust初次体验  </h2>
        <p> 2022.02.24</p>
         <p> Rust 最早是 Mozilla 雇员 Graydon Hoare 的个人项目。从 2009 年开始，得到了 Mozilla 研究院的资助。2010 年项目对外公布，2010 ～ 2011 年间实现自举。自此以后，Rust 在设计变化 -> 崩溃的边缘反复横跳（历程极其艰辛）。终于，在 2015 年 5 月 15 日发布 1.0 版。在此研发过程中，Rust 建立了一个强大且活跃社区，形成了一整套完善稳定的项目贡献机制（Rust 能够飞速发展，与这一点密不可分）。Rust 现在由 Rust 项目开发者社区 维护。
         </p>
    </a>
    <a class="a-item"  href="#/article/20211017/README.md">
        <h2>  node + crunch 实现批量图片无损压缩  </h2>
        <p> 2021.10.17</p>
         <p> 公司的SDK业务需要做包体size优化，首先瞄准图片资源
            一个含有图片资源的文件夹作为输入，压缩其中其中的图片资源 ，输出一个完全相同的文件夹
            <br/>
            * crunch  做压缩<br/>
            * node做文件名检测，递归资源复制<br/>
            * shell 做命令行封装<br/>
         </p>
        
    </a>
    <a class="a-item"  href="#/article/20210418/README.md">
        <h2>  Swift 自动闭包@autoclosure  </h2>
        <p> 2021.04.18</p>
         <p> 如下为官方文档的定义，但是建议你忽略它，搞懂了自动闭包再来看才容易懂：）<br/>
         > 自动闭包是一种自动创建的闭包，用于包装传递给函数作为参数的表达式。这种闭包不接受任何参数，当它被调
         用的时候，会返回被包装在其中的表达式的值。这种便利语法让你能够省略闭包的花括号，用一个普通的表达式来代替显式的闭包。<br/>
         这是一个正常的闭包的定义和调用:
         </p>
    </a>
    <a class="a-item"  href="#/article/20210125/README.md">
        <h2>  iOS奔溃日志分析  </h2>
        <p> 2021.01.25</p>
         <p> iOS下App如果奔溃了，可以通过奔溃日志文件来定位崩溃的问题。下面主要来分析日志文件各个字段代表的是什么含义。<br/>
            首先来看一个奔溃的日志信息：<br/>
            Incident Identifier: xxxxx<br/>
            CrashReporter Key:   bb11a137ea80e04abbab9b4385c7991e0d299599<br/>
            Hardware Model:      iPhone10,2<br/>
            Process:             xxx [476]<br/>
         </p>
    </a>
    <a class="a-item"  href="#/article/20210117/README.md">
        <h2>  iOS Crash 捕获及堆栈符号化  </h2>
        <p> 2021.01.17</p>
         <p> 最近在做 Crash 分析方面的工作，发现 iOS 的崩溃捕获和堆栈符号化虽然已经有很多资料可以参考，但是没有比较完善的成套解决方案，导致操作起来还是要踩很多坑，耽误了很多时间。所以想做一个总结，阐述 Crash 收集分析的整体思路和出坑指南，具体细节实现会给出相关参考资料。有了思路，实现也就 So Easy 啦。
         </p>
    </a>
    <a class="a-item"  href="#/article/20201021/README.md">
        <h2>  Swift 面向协议编程  </h2>
        <p> 2020.10.21</p>
         <p> 
            在WWDC15上，苹果宣布Swift是世界上第一门面向协议编程(POP)语言。相比与传统的面向对象编程 (OOP)，POP 显得更加灵活。RxSwift、ReactorKit 核心也是面向协议编程的。<br/>
            ## 一、什么是POP<br/>
            要弄清楚什么是面向协议(POP),我们应该先知道什么是Swift协议？<br/>
            我们定义一个简单的Swift协议如下：
         </p>
    </a>
    <a class="a-item"  href="#/article/202006010/README.md">
        <h2>  Node Mysql 爬取页面数据  </h2>
        <p> 2020.06.10</p>
         <p> 
            利用node爬取简单页面数据并利用本地mysql存储<br/>
            俗话说，工欲善其事，必先利其器，那么前期准备无非便是<br/>
            * [Node.js v12.16.0 文档](http://nodejs.cn/api/)<br/>
            关于node这份文档，咱们直接读中文文档也行，我觉得
         </p>
    </a>
    <a class="a-item"  href="#/article/20200519/README.md">
        <h2>  Swift 一些特性  </h2>
        <p> 2020.05.19</p>
         <p> 
            Swift 内存安全检查：当两个变量访问同一块内存时，会产生独占内存访问限制。<br/>
            发生读写权限冲突的情况：<br/>
            1. inout 参数读写冲突<br/>
            2. 结构体中函数修改成员属性读写冲突<br/>
            3. 值类型属性读写冲突
         </p>
    </a>
    <a class="a-item"  href="#/article/20200420/README.md">
        <h2>  Swift 常用第三方框架记录  </h2>
        <p> 2020.04.20</p>
         <p> 
            [Alamofire](https://github.com/Alamofire/Alamofire) :http网络请求事件处理的框架。<br/>
            [Moya](https://github.com/Moya/Moya):这是一个基于Alamofire的更高层网络请求封装抽象层。<br/>
            [Reachability.swift](https://github.com/ashleymills/Reachability.swift):用来检查应用当前的网络连接状况。
         </p>
    </a>
    <a class="a-item"  href="#/article/20190721/README.md">
        <h2>  CocoaPods使用与原理  </h2>
        <p> 2019.07.21</p>
         <p> 
            是开发iOS或MacOS应用时使用的依赖管理工具，我们在平时开发中为了提高效率，经常会集成一些第三方框架；cocoaPod的作用是可以很方便的查找到第三方框架，并为我们省去了集成第三方库所要做的额外工程配置工作，同时可以很方便的管理第三方库的后续版本升级；<br/>
            这篇文章会以介绍cocoapods的使用为基础，并对其原理部分做补充介绍；希望能加深你对cocoapods的理解，以便在日常开发中更好的使用这个工具；
         </p>
    </a>
    <a class="a-item"  href="#/article/20190213/README.md">
        <h2>  iOS JenKins + fastlane 实现自动打包及持续集成  </h2>
        <p> 2019.02.13</p>
         <p> 
            持续集成是一种软件开发实践：许多团队频繁地集成他们的工作，每位成员通常进行日常集成，进而每天会有多种集成。每个集成会由自动的构建（包括测试）来尽可能快地检测错误。<br/>
            许多团队发现这种方法可以显著的减少集成问题并且可以使团队开发更加快捷。
         </p>
    </a>
    <a class="a-item"  href="#/article/20190102/README.md">
        <h2>  YYModel源码分析  </h2>
        <p> 2019.01.02</p>
         <p> 
            YYModel是一个序列化和反序列化库，说得更直白一点就是将NSString,NSData,NSDictionary形式的数据转换为具体类型的对象，以及将具体类型的对象转换为NSDictionary。<br/>
            我们在没有开始看源码之前，我们猜测下如果这件事情要做应该怎么做？
         </p>
    </a>
    <a class="a-item"  href="#/article/20181229/README.md">
        <h2>  UIViewController 生命周期  </h2>
        <p> 2018.12.29</p>
         <p> 
            事件发生的须序非常重要，这好让程序员能在适当的时机执行事件，此时了解view life Cycle是非常必要的。 <br>
            首先访问view属性，如果view存在，则直接加载。如果不存在，则调用loadView方法。 <br>
            loadView方法执行操作如下：
         </p>
    </a>
    <a class="a-item"  href="#/article/20181211/README.md">
        <h2>  GCC 常用技巧  </h2>
        <p> 2018.12.11</p>
         <p> 
            Clang 是一个 C++ 编写，基于 LLVM 的 C/C++、Objective-C 语言的轻量级编译器，在 2013.04 开始，已经全面支持 C++11 标准。<br/>
            \#pragma 宏定义在本质上是声明，常用的功能就是注释，尤其是给 Code 分段注释；另外，还支持处理编译器警告。<br/>
            #pragma clang diagnostic ignored "-Wdeprecated-declarations"
         </p>
    </a>
    <a class="a-item"  href="#/article/20181020/README.md">
        <h2>  iOS Association 关联对象  </h2>
        <p> 2018.10.20</p>
         <p> 
            默认情况下，由于分类底层结构的限制，不能直接给 Category 添加成员变量，但是可以通过关联对象间接实现 Category 有成员变量的效果。<br/>
            #import "Person.h"<br/>
            @interface Person (Test)<br/>
            @property (nonatomic, assign) int height;<br/>
            @end
         </p>
    </a>
    <a class="a-item"  href="#/article/20180921/README.md">
        <h2>  iOS @property属性详解  </h2>
        <p> 2018.09.21</p>
         <p> 
            property 声明了ivar（实例变量）和存取方法（access method ＝ getter + setter)<br/>。所有关键词都是用于声明或者控制这些。其主要的作用就在于封装对象中的数据。 Objective-C <br/>对象通常会把其所需要的数据保存为各种实例变量。实例变量一般通过getter来访问，setter用于写入变量值。
         </p>
    </a>
    <a class="a-item"  href="#/article/20180920/README.md">
        <h2>  iOS图像渲染 </h2>
        <p> 2018.09.20</p>
         <p> 
            UIKit是iOS开发者最常用和熟悉的UI框架，承担着UI显示和用户交互的功能。UIView在UIKit框架中最基本的单元，[苹果官方文档](https://developer.apple.com/documentation/uikit/uiview?language=objc)描述UIView的基本功能：<br/>
            * 绘制和动画(Drawing and animation)<br/>
            * 布局和子视图管理（Layout and subview management)<br/>
            * 事件处理（Event handling）
         </p>
    </a>
    <a class="a-item"  href="#/article/20180706/README.md">
        <h2>  iOS App Store 被拒总结 </h2>
        <p> 2018.07.06</p>
         <p> 
            这个被拒的比较常见。个人总结主要是因为在每年苹果发布会结束的一段时间里面，审核会比较严格。在这段时间里面提交app名称里面要是保函了关键词，肯定被拒。所以想在app信息的名称里面添加关键词的最好在这段时候里面先去掉，之后再加回去。个人建议是不要在这个地方添加关键词，苹果已经说明了这个地方加关键词是起不到搜索作用的。
         </p>
    </a>
    <a class="a-item"  href="#/article/20180508/README.md">
        <h2>  iOS WKWebView简单使用  </h2>
        <p> 2018.05.08</p>
         <p> 
            - 允许JavaScript的Nitro库加载并调用(UIWebView中限制) <br/>
            - 支持更多的H5特性<br/>
            - 将UIWebViewDelegate与UIWebView重构成了14个类与3个协议
         </p>
    </a>
    <a class="a-item"  href="#/article/20180417/README.md">
        <h2>  iOS 数据存储总结  </h2>
        <p> 2018.04.17</p>
         <p> 
            - NSKeyedArchiver： 采用归档的形式来保存数据沙盒中； <br/>
            - NSUserDefaults：偏好设置数据存到沙盒的Library/Preferences目录（本质是plist）; <br/>
            - Write写入方式： 永久保存在磁盘中; <br/>
            - 采用SQLite等数据库来存储数据。 <br/>
         </p>
    </a>
    <a class="a-item"  href="#/article/20180201/README.md">
        <h2>  iOS 导航栏的正确隐藏方式  </h2>
        <p> 2018.02.01</p>
         <p> 
            缺点: 这样做有一个缺点就是在切换tabBar的时候有一个导航栏向上消失的动画<br/>
            - (void)viewWillAppear:(BOOL)animated {<br/>
             [super viewWillAppear:animated];<br/>
             [self.navigationController setNavigationBarHidden:YES animated:YES];<br/>
            }
         </p>
    </a>
</div>




