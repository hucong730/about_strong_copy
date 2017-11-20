# about_strong_copy
关于Objective-C里类的属性修饰词strong和copy的一些区别。

>首先在控制器里面定义一个copy的string，
```objective-c
@interface ViewController ()
@property (nonatomic, copy) NSString *string;
@end
```
>然后定义一个可变的字符串，把可变的字符串赋值给self.string，然后修改可变字符串，发现self.string还是修改之前的值。
```objective-c
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    NSMutableString *mStr = [NSMutableString stringWithString:@"hello"];
    self.string = mStr;
    [mStr appendString:@", world"];
    NSLog(@"mStr = %p, self.string = %p", mStr, self.string);
    NSLog(@"mStr = %@, self.string = %@", mStr, self.string);
    
}

//打印结果如下：
//mStr = 0x60000025b6f0, self.string = 0xa00006f6c6c65685
//mStr = hello, world, self.string = hello
```
>从上面可以看到`self.string`和`mStr`的地址不一样，说明`mStr`被深拷贝，然后`self.string`指向被深拷贝的那个地址。所以你修改`mStr`的值不会影响`self.string`的值。
>然后进一步探究其原理，把`self.string = mStr;`改成`_string = mStr;`之后，发现`self.string`的值被`mStr`改了。
```objective-c
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    NSMutableString *mStr = [NSMutableString stringWithString:@"hello"];
    _string = mStr;
    [mStr appendString:@", world"];
    NSLog(@"mStr = %p, self.string = %p", mStr, self.string);
    NSLog(@"mStr = %@, self.string = %@", mStr, self.string);
}

//打印结果如下：
//mStr = 0x60c0006457c0, self.string = 0x60c0006457c0
//mStr = hello, world, self.string = hello, world
```

>我们发现`self.string`和`mStr`的地址一样，所以被改了，就很正常不过了，再仔细一想，`_string = mStr;`和`self.string = mStr;`到底有啥区别呢？其实就是`_string`就是正常的成员变量赋值，而`self.string`就不一样了，他是走`set`方法。为了验证是`set`方法导致的，我们重写`set`方法加以验证。
```objective-c
@interface ViewController ()

@property (nonatomic, copy) NSString *string;

@end

@implementation ViewController

- (void)setString:(NSString *)string {
    _string = string;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    NSMutableString *mStr = [NSMutableString stringWithString:@"hello"];
    self.string = mStr;
    [mStr appendString:@", world"];
    NSLog(@"mStr = %p, self.string = %p", mStr, self.string);
    NSLog(@"mStr = %@, self.string = %@", mStr, self.string);
}
@end

//打印结果如下：
//mStr = 0x608000043f30, self.string = 0x608000043f30
//mStr = hello, world, self.string = hello, world
```
>此时，两者的值都是一样的，说明在带有`copy`修饰的属性，编译器自动帮你生成的`set`方法是把传入的参数调用了`copy`方法，为了验证这个猜想，我们模拟的写一下他的`set`方法，
```objective-c
- (void)setString:(NSString *)string {
    _string = [string copy];
}

//打印结果如下：
//mStr = 0x608000252a20, self.string = 0xa00006f6c6c65685
//mStr = hello, world, self.string = hello
```

>这样我们就明白了`copy`和`strong`修饰的属性到底被编译器干了什么，`strong`修饰其实就是简单的引用赋值(c的指针传递)，而`copy`修饰的就是调用了`set`方法传入的形参的`copy`方法。
```objective-c
- (void)setString:(NSString *)string {
    _string = [string copy]; //copy修饰
}
```objective-c
- (void)setString:(NSString *)string {
    _string = string; //strong修饰
}
```

>补充说明：
关于copy和mutableCopy哪个是深拷贝，哪个是浅拷贝，其实说到底就是到底生成了新对象没？所以通过打印前后的地址就能知道结果。然后总结一下，NSString、NSArray、NSDictionary等这些这些不可变对象进行了copy，是浅拷贝。其他（NSString、NSArray、NSDictionary等mutableCopy，NSMutableString、NSMutableArray等的copy、mutableCopy）都是深拷贝。

