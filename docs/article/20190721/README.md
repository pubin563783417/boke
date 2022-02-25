<!-- README.md -->

# CocoaPods使用与原理

2019-07-21

## 前言
[CocoaPods](https://cocoapods.org/) 是开发iOS或MacOS应用时使用的依赖管理工具，我们在平时开发中为了提高效率，经常会集成一些第三方框架；cocoaPod的作用是可以很方便的查找到第三方框架，并为我们省去了集成第三方库所要做的额外工程配置工作，同时可以很方便的管理第三方库的后续版本升级；

这篇文章会以介绍cocoapods的使用为基础，并对其原理部分做补充介绍；希望能加深你对cocoapods的理解，以便在日常开发中更好的使用这个工具；

## Cocoapods安装

[可以看这里](https://zhaoxin.pro/14916226270738.html)

## CocoaPods的使用

在安装了cocoapods的前提下，只需要cd到你工程的根目录，然后执行以下命令

```
pod init
```

这个命令会在工程根目录下生成一个Podfile文件； Podfile 文件用于定义项目所需要使用的第三方库，使用Ruby语法书写。该文件支持高度定制，你可以根据个人喜好对其做出定制；

以下是一个简单的Podfile示例：

```
platform :ios, '9.0'

target 'TestPod' do
    pod 'SDWebImage/Core', '4.4.3'
    pod 'YYModel', '1.0.4'
    pod 'CocoaLumberjack', '~> 1.8.1'
    pod 'FMDB', '~> 2.2'
end
```

更多写法可以看官方 [Podfile 指南] (http://guides.cocoapods.org/syntax/podfile.html)

然后再通过执行pod install就可以把这些开源组件集成进APP了。安装完，这些组件都放在一个Pods的工程中，之后会用xcode的workspace来管理这个工程和你自己的工程。以后就通过打开xxx.xcworkspace来运行工程就行了；

第一次运行pod install时, .xcworkspace项目和Pods目录还不存在, pod install命令会创建.xcworkspace和Pods目录；

## pod install 与 pod update 区别

我们在使用这个工具时，最经常用到的命令就是pod install 和 pod update；这两个命令都能为我们安装指定的Pod库，但是他们的使用是有区别的，有各自不同的作用和适用场景；

了解这两个命令的差异前，需要先对Podfile.lock文件有所了解；

- Podfile.lock文件中记录了每个pod的当前已安装版本，并锁定这些版本(.lock命名因此而来)；
- 当运行pod install命令时，它只解析尚未列在Podfile.lock中的依赖库，在下载并安装新的pod时, 会在Podfile.lock文件中写入新的pod库的当前指定版本；
- 对于已经在Podfile.lock中列出的pod, pod install命令不会尝试检查是否有更新的版本；而是直接使用在Podfile.lock文件中的pod版本；

## pod install

在项目中第一次使用CocoaPods时，安装指定的pod库需要使用这个命令；
当在Podfile中 增加 或 删除 某个pod后, 使用这个命令进行更新；pod install不会尝试更新已安装的pod的版本；

### pod update

1、当运行pod update时，CocoaPods会把Podfile中所有的pod都更新到最新版本；并在Podfile.lock中写入最新的版本号；
2、执行pod update PODNAME命令更新指定的某个PODNAME库到最新版本；

### pod outdated

pod outdated命令可以检测出所有比Podfile.lock中写入的版本（每个pod当前安装的版本）更新的pod版本；

## 创建自己的CocoaPods库

### 1、podspec文件

.podspec 也是一个文件，该文件描述了一个库是怎样被添加到工程中的。它支持的功能有：列出源文件、framework和lib、编译选项和某个库所需要的依赖等。

通过~/.cocoapods/repos路径查看 本地中心仓库文件夹，会发现保存的都是podspec文件；当我们执行pod search PODNAME命令时，cocoapods会在它的 本地中心仓库文件夹 查找已有的podspec文件，如果匹配到PODNAME库，就会列出所有release 版本；

### 2、为开源项目创建podspec文件

我们可以为自己的开源项目创建podspec文件，在工程目录下通过如下命令初始化一个podspec文件：

```
pod spec create your_spec_name
end
```

执行该命令后，CocoaPods 会生成一个名为your_spec_name.podspec的文件，然后我们修改其中的相关内容即可，如下是一个例子：

```
Pod::Spec.new do |s|
  s.name             = 'QLMutilTableView'
  s.version          = '1.0.0'
  s.summary      = '多表的实现'
  s.homepage    = 'https://www.jianshu.com/p/589fcc1d22b9'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'qianlei' => '13698090397@163.com' }
  s.source           = { :git => '//https://github.com/qianleileilei/QLMutilTableView.git', :tag => 1.0.0 }
  s.ios.deployment_target = '9.0'
  s.source_files = 'QLMutilTableView/QLMutilTableView/**/*'
  s.frameworks = 'UIKit', 'Foundation'

end
```

之后可以通过如下命令验证podspec文件是否有错误或警告：

```
pod spec lint your_podspec_name.podspec
```

如果验证不通过，会有详细的ERROR和WARING提示，可根据提示信息对文件进行修改；验证通过后，使用以下命令把podspec文件推送到Cocoapods：

```
pod trunk push your_podspec_name.podspec
```

这一步完成后，并在更新本地的repo中心仓库后，就可以通过pod search PodName查看到你的开源库信息了；第三方使用者也可以通过在Podfile文件中添加pod 'PodName'直接引用到你的开源库了；

### 3、添加git仓库作为私有中心仓库

直接在~/.cocoapods/repo目录下面创建一个目录YourPrivatePodsName，放自己的podspec，这里需要符合cocoapods对中心代码库的文件结构树的规定：

- 在~/.cocoapods/repo 目录下创建 YourPrivatePodsName
- 在YourPrivatePodsName文件夹下创建 YourProjrctName
- 在YourProjrctName文件夹下创建 1.0.0版本号
- 在1.0.0版本文件夹下放入 对应的podspec文件
- 把YourPrivatePodsName用git托管，推送到git server上

这个时候就可以共享你自己的私有中心仓库了，其他人如果更新了本地repo仓库（通过以下命令添加了repo仓库）：

```
pod repo add reponame git@example.com: YourPrivatePodsName.git
```

就可以通过pod search YourProjrctName搜索到共享的开源库了；

## Cocoapods原理

CocoaPods 会把所有的依赖库都放到一个名为 Pods 项目中，之后通过xcode的workspace来管理pod工程和你自己的工程；

Pods 项目最终会编译成一个名为 libPods.a 的文件，主项目只需要依赖这个 .a 文件即可；

在看一下pod install的执行流程：

- 查看 ~/.cocoapods/repo/master/Specs 是否存在当前需要的依赖
- 存在，从这个本地三方库信息库中获取 Podfile 中对应三方库的 git 地址
- 不存在，输出 Setting up CocoaPods Master repo，并拉取三方库信息库到 ~/.cocoapods/repo/中
- 使用 git 命令从 GitHub 上拉取 Podfile 中对应的三方库源码
- 
关于pod install过程更深入的理解还可以看[这里](https://objccn.io/issue-6-4/)

## 更新pod本地仓库

```
pod repo update
pod install --repo-update
```


注意：pod repo update 与 pod update [依赖名] 的区别

前段时间，运行pod install安装指定版本，提示需要使用pod repo update更新本地仓库，具体如下：

```
$ pod install
Analyzing dependencies
[!] CocoaPods could not find compatible versions for pod "FitCloudKit":
  In Podfile:
    FitCloudKit (= 1.1.9)

None of your spec sources contain a spec satisfying the dependency: `FitCloudKit (= 1.1.9)`.

You have either:
 * out-of-date source repos which you can update with `pod repo update` or with `pod install --repo-update`.
 * mistyped the name or version.
 * not added the source repo that hosts the Podspec to your Podfile.

Note: as of CocoaPods 1.0, `pod repo update` does not happen on `pod install` by default.
```
