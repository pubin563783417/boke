<!-- README.md -->

# 数据存储总结

2018-04-17

## 存储方式介绍

- NSKeyedArchiver： 采用归档的形式来保存数据沙盒中；
- NSUserDefaults：偏好设置数据存到沙盒的Library/Preferences目录（本质是plist）;
- Write写入方式： 永久保存在磁盘中;
- 采用SQLite等数据库来存储数据。

### NSKeyedArchiver 归档

 采用归档的形式来保存数据，该数据对象需要遵守NSCoding协议，并且该对象对应的类必须提供encodeWithCoder:和initWithCoder:方法。
前一个方法告诉系统怎么对对象进行编码，而后一个方法则是告诉系统怎么对对象进行解码。

缺点：
归档的形式来保存数据，只能一次性归档保存以及一次性解压。所以只能针对小量数据，而且对数据操作比较笨拙，即如果想改动数据的某一小部分，还是需要解压整个数据或者归档整个数据。


```c
//解档
 2   - (id)initWithCoder:(NSCoder *)aDecoder {
 3       if ([super init]) {
 4           self.avatar = [aDecoder decodeObjectForKey:@"avatar"];
 5           self.name = [aDecoder decodeObjectForKey:@"name"];
 6           self.age = [aDecoder decodeIntegerForKey:@"age"];
 7       }
 8       return self;
 9   }
10   //归档
11   - (void)encodeWithCoder:(NSCoder *)aCoder {
12       [aCoder encodeObject:self.avatar forKey:@"avatar"];
13       [aCoder encodeObject:self.name forKey:@"name"];
14       [aCoder encodeInteger:self.age forKey:@"age"];
15   }
```

## NSUserDefaults 用户偏好

用来保存应用程序设置和属性、用户保存的数据。用户再次打开程序或开机后这些数据仍然存在。
NSUserDefaults可以存储的数据类型包括：NSData、NSString、NSNumber、NSDate、NSArray、NSDictionary。如果要存储其他类型，则需要转换为前面的类型，才能用NSUserDefaults存储。

```c
//1.获得NSUserDefaults文件
NSUserDefaults *userDefaults = [NSUserDefaults standardUserDefaults];
//2.向文件中写入内容
[userDefaults setObject:@"AAA" forKey:@"a"];
[userDefaults setBool:YES forKey:@"sex"];
[userDefaults setInteger:21 forKey:@"age"];
//2.1立即同步
[userDefaults synchronize];
//3.读取文件
NSString *name = [userDefaults objectForKey:@"a"];
BOOL sex = [userDefaults boolForKey:@"sex"];
NSInteger age = [userDefaults integerForKey:@"age"];
NSLog(@"%@, %d, %ld", name, sex, age);
```

## NSDict NSArray 的Write方式

由系统提供的方法快速直接的把基本数据写入到磁盘，优点是便捷，缺点是缺少中间控制

写入:

```c
NSArray *array = @[@"123", @"456", @"789"];
[array writeToFile:fileName atomically:YES]; 
```

读取:

``` c
NSString *path = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject;
NSString *fileName = [path stringByAppendingPathComponent:@"123.plist"];
NSArray *result = [NSArray arrayWithContentsOfFile:fileName];
```

## SQL 采用数据库来存储数据

用数据库方式来存储，使用SQL语句，跨平台，适用于大量的同类型数据

一般有这些 :

* SQLLite : 由系统提供原生支持
* Realm : 需要引入Realm库


