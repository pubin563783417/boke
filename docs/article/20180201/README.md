<!-- README.md -->

# iOS导航栏的正确隐藏方式

2018-02-01

## 第一种做法：

缺点: 这样做有一个缺点就是在切换tabBar的时候有一个导航栏向上消失的动画

```c
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];

    [self.navigationController setNavigationBarHidden:YES animated:YES];
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];

    [self.navigationController setNavigationBarHidden:NO animated:YES];
}
```

## 第二种做法：

```c
@interface MyPageViewController () <UINavigationControllerDelegate>

@end

@implementation MyPageViewController 
- (void)viewDidLoad {
    [super viewDidLoad];

    // 设置导航控制器的代理为self
    self.navigationController.delegate = self;
}

#pragma mark - UINavigationControllerDelegate
// 将要显示控制器
- (void)navigationController:(UINavigationController *)navigationController willShowViewController:(UIViewController *)viewController animated:(BOOL)animated {
    // 判断要显示的控制器是否是自己
    BOOL isShowHomePage = [viewController isKindOfClass:[self class]];

    [self.navigationController setNavigationBarHidden:isShowHomePage animated:YES];
}
```


