# iOS常见面试题总结

## OC底层基础
### 1:分类和扩展有什么区别？可以分别用来做什么？分类有哪些局限性？分类的结构体里面有哪些成员？
### 
- 分类主要用来为某个类添加方法、属性、协议（一般用来为系统的类拓展方法或者把某个复杂类按照功能拆到不同的文件里）
- 扩展主要用来为某个类原来没有的成员变量、属性、方法。注：方法只是声明（我一般用来扩展声明私有属性，或把.h的只读属性重写或写可读的）
**分类和扩展的区别**
- 分类是在运行时把分类信息合并到类信息中，扩展是在编译时就合并到类中的
- 分类声明的属性只会生成setter/getter方法的声明，不会自动生成成员变量和getter/setter方法的实现，而扩展会
- 分类不可用为类添加实例变量，拓展可以
- 分类可以为类添加方法的实现，而扩展只能声明方法，不能实现
**分类的局限性**
- 无法为类添加实例变量，但可以通过关联对象进行实现，注：关联对象中内存管理没有weak，用的时候需要注意野指针的问题，可以通过其他办法实现
- 分类的方法若干和类中原本的实现重名，会覆盖原本方法的实现，（并不是真正的覆盖）
- 多个分类的方法名重名，会调用最后编译的那个分类的实现
**分类的结构体有哪些成员**

```struct category_t {
    const char *name; //名字
    classref_t cls; //类的引用
    struct method_list_t *instanceMethods;//实例方法列表
    struct method_list_t *classMethods;//类方法列表
    struct protocol_list_t *protocols;//协议列表
    struct property_list_t *instanceProperties;//实例属性列表
    // 此属性不一定真正的存在
    struct property_list_t *_classProperties;//类属性列表
};
```
### 2:HTTPS和HTTP的区别 ### 

**HTTPS协议 = HTTP协议+SSL/TLS协议**
- ssl全称是secure sockets layer，即安全套接层协议，是为网络通信提供安全及数据完整性的一种安全协议。
- tls的全称是 transport layer security，即安全传输层协议
**即https是安全的http**
- https，全称是 hyper text transfer protocol secure，相比http，多了一个secure，这个secure是怎么来的呢？是由 tls（ssl）提供的！大概就是一个叫openssl的library提供的。https和http都是属于application layer，基于tcp（以及udp）协议，但又完全不一样，tcp的port是80，https用的是443（google发明的一个新协议，叫QUIC，并不基于tcp，用的port也是443，同样是用来给https的）。总结：https和http类似，但比http安全。
- https协议需要到CA申请证书，一般免费证书很少
- http是超文本传输协议，信息是明文传输，http则是具有安全性的ssl加密传输协议
- http的连接很简单，是无状态的
### **3:讲一下atomic的实现机制；为什么不能保证绝对的线程安全（最好可以结合场景来说）？**
### 
**atomic的实现机制**

- atomic是property的修饰词之一，表示是原子性的，使用方式为@property(atomic)int age;此时编译器会自动生成 getter/setter 方法，最终会调用objc_getProperty和objc_setProperty方法来进行存取属性。

- 若此时属性用atomic修饰的话，在这两个方法内部使用os_unfair_lock 来进行加锁，来保证读写的原子性。锁都在PropertyLocks 中保存着（在iOS平台会初始化8个，mac平台64个），在用之前，会把锁都初始化好，在需要用到时，用对象的地址加上成员变量的偏移量为key，去PropertyLocks中去取。因此存取时用的是同一个锁，所以atomic能保证属性的存取时是线程安全的。

注：由于锁是有限的，不用对象，不同属性的读取用的也可能是同一个锁

**atomic为什么不能保证绝对的线程安全？**

- atomic在getter/setter方法中加锁，仅保证了存取时的线程安全，假设我们的属性是@property(atomic)NSMutableArray *array;可变的容器时,无法保证对容器的修改是线程安全的.

- 在编译器自动生产的getter/setter方法，最终会调用objc_getProperty和objc_setProperty方法存取属性，在此方法内部保证了读写时的线程安全的，当我们重写getter/setter方法时，就只能依靠自己在getter/setter中保证线程安全

### 4:Autoreleasepool所使用的数据结构是什么？AutoreleasePoolPage结构体了解么？
### 
- Autoreleasepool是由多个AutoreleasePoolPage以双向链表的形式连接起来的.

- Autoreleasepool的基本原理：在每个自动释放池创建的时候，会在当前的AutoreleasePoolPage中设置一个标记位，在此期间，当有对象调用autorelsease时，会把对象添加 AutoreleasePoolPage中

- 若当前页添加满了，会初始化一个新页，然后用双向链表链接起来，并把新初始化的这一页设置为hotPage,当自动释放池pop时，从最下面依次往上pop，调用每个对象的release方法，直到遇到标志位。

AutoreleasePoolPage结构如下

```
class AutoreleasePoolPage {
    magic_t const magic;
    id *next;//下一个存放autorelease对象的地址
    pthread_t const thread; //AutoreleasePoolPage 所在的线程
    AutoreleasePoolPage * const parent;//父节点
    AutoreleasePoolPage *child;//子节点
    uint32_t const depth;//深度,也可以理解为当前page在链表中的位置
    uint32_t hiwat;
}
```
### 5:TCP为什么要三次握手，四次挥手？
### 
**三次握手**
- 客户端想服务端发起请求链接，首先发送syn报文，syn=1，seq=x，并且向客户端进入syn_sent状态
- 服务端收到请求链接，向客户端进行回复，并发送响应报文，syn=1，seq=y，ack=1，ack=x+1，并且服务端进入到syn_rcvd状态
- 客户端收到确认报文后，向服务端发送确认报文，ack=1，ack=y+1，此时客户端进入到 established，服务端收到用户端发送过来的确认报文后，也进入到 established 状态，此时链接成功
**四次挥手**

- 客户端向服务端发送关闭链接，并停止发送数据
- 服务端收到关闭链接请求，向客户端发送回应，我知道了，然后停止接收数据
- 当服务端发送数据结束之后，向客户端发起关闭链接，并停止发送数据
- 客户端收到关闭链接的请求时，向服务端发送回应，我知道了，然后停止接收数据
**为什么需要三次握手**

- 为了防止已失效的连接请求报文突然又传到了服务端，因而产生错误，假定这是一个早已经失效的报文段。但server收到此失效的连接请求报文段后，就误认为是client再次发出一个新的连接请求，于是就向client发出确认报文段，同意建立连接。假设不采用“三次握手”，那么只要server发出确认，新的连接就建立了。由于现在client并没有发出建立连接的请求，因此不会理睬server的确认，也不会向server发送数据，但server以为新的运输连接已经建立，并且一直等待clenet发送数据。这样server的资源就白白浪费了
**为什么四次挥手**

因为tcp是双全工通信的，在接收到客户端的关闭请求时，还可能向客户端发送数据，因此不能再回应关闭请求连接的同时发送关闭连接的请求
### 6:对称加密和非对称加密的区别？分别有哪些算法的实现？
对称加密，加密和解密可以使用同一密钥
 
 - 非对称加密，使用一对密钥用于加密和解密，分别为公开密钥和私有密钥。公开密钥所有人都可以获得，通信发送方获得接收方的公开密钥后，就可以使用公开密钥进行加密，接收方收到通信内容后使用私有密钥解密。
 - 对称加密常用的算法实现有AES,ChaCha20,DES,不过DES被认为是不安全的，非对称加密的算法实现有RSA,ECC
 
 iOS加密相关算法框架：`CommonCrypto`。

**1:对称加密： DES、3DES、AES**

- 加密和解密使用同一个密钥。

- 加密解密过程：

明文->密钥加密->密文，

密文->密钥解密->明文。

优点：算法公开、计算量少、加密速度快、加密效率高、适合大批量数据加密；

缺点：双方使用相同的密钥，密钥传输的过程不安全，易被破解，因此为了保密其密钥需要经常更换。

> AES：AES又称高级加密标准，是下一代的加密算法标准，支持128、192、256位密钥的加密，加密和解密的密钥都是同一个。iOS一般使用ECB模式，16字节128位密钥。

AES算法主要包括三个方面：轮变化、圈数和密钥扩展。

优点：高性能、高效率、灵活易用、安全级别高。

缺点：加密与解密的密钥相同，所以前后端利用AES进行加密的话，如何安全保存密钥就成了一个问题。

> DES：数据加密标准，DES算法的入口参数有三个：Key、Data、Mode。

其中Key为7个字节共56位，是DES算法的工作密钥；Data为8个字节64位，是要被加密或被解密的数据；Mode为DES的工作方式，有两种：加密、解密。

缺点：与AES相比，安全性较低。

> 3DES：3DES是DES加密算法的一种模式，它使用3条64位的密钥对数据进行三次加密。是DES向AES过渡的加密算法，是DES的一个更安全的变形。它以DES为基本模块，通过组合分组方法设计出分组加密算法。

> 2.非对称加密:RSA加密

- 非对称加密算法需要成对出现的两个密钥，公开密钥(publickey) 和私有密钥(privatekey) 。
**加密解密过程：**对于一个私钥，有且只有一个与之对应的公钥。生成者负责生成私钥和公钥，并保存私钥，公开公钥。
公钥加密，私钥解密；或者私钥数字签名，公钥验证。公钥和私钥是成对的，它们互相解密。

特点：
- 1). **对信息保密，防止中间人攻击：**将明文通过接收人的公钥加密，传输给接收人，因为只有接收人拥有对应的私钥，别人不可能拥有或者不可能通过公钥推算出私钥，所以传输过程中无法被中间人截获。只有拥有私钥的接收人才能阅读。此方法通常用于交换对称密钥。
- 2). **身份验证和防止篡改：**权限狗用自己的私钥加密一段授权明文，并将授权明文和加密后的密文，以及公钥一并发送出来，接收方只需要通过公钥将密文解密后与授权明文对比是否一致，就可以判断明文在中途是否被篡改过。此方法用于数字签名。
- **优点：**加密强度小，加密时间长，常用于数字签名和加密密钥、安全性非常高、解决了对称加密保存密钥的安全问题。
- **缺点：**加密解密速度远慢于对称加密，不适合大批量数据加密。

3. 哈希算法加密：MD5加密、.SHA加密、HMAC加密

哈希算法加密是通过哈希算法对数据加密，加密后的结果不可逆，即加密后不能再解密。
**特点：**不可逆、算法公开、相同数据加密结果一致。
**作用：**信息摘要，信息“指纹”，用来做数据识别的。如：用户密码加密、文件校验、数字签名、鉴权协议。
MD5加密：**对不同的数据加密的结果都是定长的32位字符。

**.SHA加密：**安全哈希算法，主要适用于数字签名标准(DSS)里面定义的数字签名算法(DSA)。对于长度小于2^64位的消息，SHA1会产生一个160位的消息摘要。当接收到消息的时候，这个消息摘要可以用来验证数据的完整性。在传输的过程中，数据很可能会发生变化，那么这时候就会产生不同的消息摘要。当然除了SHA1还有SHA256以及SHA512等。

**HMAC加密：**给定一个密钥，对明文加密，做两次“散列”，得到的结果还是32位字符串。

4. Base64加密

一种编码方式，严格意义上来说不算加密算法。其作用就是将二进制数据编码成文本，方便网络传输。

用 base64 编码之后，数据长度会变大，增加了大约 1/3，但是好处是编码后的数据可以直接在邮件和网页中显示；

虽然 base64 可以作为加密，但是 base64 能够逆运算，非常不安全！

base64 编码有个非常显著的特点，末尾有个 ‘=’ 号。


### 7:HTTPS的握手流程？为什么密钥的传递需要使用非对称加密？双向认证了解么？
HTTPS的握手流程，如下图，摘自图解HTTP

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7f2538898fc4f6fbe854929a193ba9f~tplv-k3u1fbpfcp-watermark.image)

1. 客户端发送 client hello 报文开始ssl通信。报文中包含客户端支持的ssl的版本，加密组件列表。
2. 服务器收到之后，会以server hello报文作为应答。和客户端一样，报文中包含客户端支持的ssl的版本，加密组件列表。服务器的加密组件内容是从接收到的客户端加密组件内筛选出来的
3. 服务器发送certificate报文。报文中包含公开密钥证书。
4. 然后服务器发送server hello done 报文通知客户端，最初阶段的ssl握手协商部分结束
5. ssl第一次握手结束之后，客户端以client key exchange报文作为会议。报文中包含通信加密中使用的一种被称为pre-master secret的随机密码串
6. 接着客户端发送change cipher space报文。该报文会提示服务器，在报文之后的通信会采用pre-master secret密钥加密
7. 客户端发送finished报文。该报文包含连接至今全部报文的整体校验值。这次握手协商是否能够成功，要以服务器是否能够正确解密该报文作为判定标准
8. 服务器同样发送change cipher space报文
9. 服务器同样发送finished报文
10. 服务器和客户端的finished报文交换完毕后，ssl连接建立完成，从此开始http通信，通信的内容都使用pre-master secret加密。然后开始发送http请求
11. 应用层收到http请求之后，发送http相应
12. 最后有客户端断开连接
**为什么密钥的传递需要使用非对称加密？**

非对称加密是为了后面客户端生成的pre-master secret密钥的安全，通过上面的步骤能够得知，服务端向客户端发送公钥证书这一步是可能被人拦截的没如果使用对称加密的话，客户端向服务端发送pre-master secret密钥的时候，被黑客拦截的话，就能够使用公钥进行解密，就无法保证pre-master secret密钥的安全了

**双向认证了解么？**

上面的https通信流程只验证了服务端的身份，而服务端没有验证客户端的身份，双向认证是服务端也要确保客户端的身份，大概流程是客户端在校验完服务器的证书之后，会向服务器发送自己的公钥，然后服务端用公钥加密产生一个新的密钥，传给客户端，客户端再用私钥解密，以后就用此密钥进行对称加密的通信

### 8:如何用Charles抓HTTPS的包？其中原理和流程是什么？
**流程：**

- 首先在手机上安装Charles证书
- 在代理设置中开启Enable SSL Proxying
- 之后添加需要抓取服务端的地址
**原理：**

Charles作为中间人，对客户端伪装成服务端，对服务端伪装成客户端。简单来说：
- 截获客户端的HTTPS请求，伪装成中间人客户端去向服务端发送HTTPS请求
- 接受服务端返回，用自己的证书伪装成中间人服务端向客户端发送数据内容。


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6e88eda9ecb4f01b4bdd7a1299b6d3d~tplv-k3u1fbpfcp-watermark.image)

### 9:什么是中间人攻击？如何避免？
- 中间人攻击就是截获到客户端的请求以及服务器的响应，比如Charles抓取HTTPS的包就属于中间人攻击。
- 避免的方式：客户端可以预埋证书在本地，然后进行证书的比较是否是匹配的

### 10:App网络层有哪些优化策略？
- 优化dns解析和缓存
- 对传输的数据进行压缩，减少传输的数据
- 使用缓存手段减少请求的发起次数
- 使用策略来减少请求的发起次数，比如在上一个请求为着地之前不进行新的请求
- 避免网络抖动，提供重发机制

### 11：[self class] 与 [super class]

```
@implementation Son : Father
- (id)init
{
    self = [super init];
    if (self)
    {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
return self;
}
@end
```
**self和super的区别**
- self是类的一个隐藏参数，每个方法的实现的第一个参数即为self
- super并不是隐藏参数，他实际上只是一个**“编译标识符”**，他告诉编译器，当调用方法时，去调用父类的方法，而不是本类的
- 在调用[super class]的时候，runtime会去调用objc_msgSendSuper方法，而不是objc_msgSend

```
OBJC_EXPORT void objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )
 
 
/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained id receiver;
 
    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
}

```
**入院考试第一题错误的原因就在这里，误认为[super class]是调用的[super_class class]。**
objc_msgSendSuper的工作原理应该是这样的:
- 从objc_super结构体指向的superClass父类的方法列表开始查找selector，
- 找到后以objc->receiver去调用父类的这个selector。注意，最后的调用者是objc->receiver，而不是super_class！
那么objc_msgSendSuper最后就转变成


```
objc_msgSend(objc_super->receiver, @selector(class))
 
+ (Class)class {
    return self;
}
```

### 12：property和属性修饰符
@property的本质是 ivar(实例变量) + setter + getter.

我们每次增加一个属性时内部都做了什么：

1.系统都会在 ivar_list 中添加一个成员变量的描述；

2.在 method_list 中增加 setter 与 getter 方法的描述；

3.在属性列表中增加一个属性的描述；

4.然后计算该属性在对象中的偏移量；

5.给出 setter 与 getter 方法对应的实现，在 setter 方法中从偏移量的位置开始赋值，在 getter 方法中从偏移量开始取值，为了能够读取正确字节数，系统对象偏移量的指针类型进行了类型强转。

修饰符：

MRC下： assign、retain、copy、readwrite、readonly、nonatomic、atomic等。

ARC下：assign、strong、weak、copy、readwrite、readonly、nonatomic、atomic、nonnull、nullable、null_resettable、_Null_unspecified等。

下面分别解释

assign：用于基本数据类型，不更改引用计数。如果修饰对象(对象在堆需手动释放内存，基本数据类型在栈系统自动释放内存)，会导致对象释放后指针不置为nil 出现野指针。

retain：和strong一样，释放旧对象，传入的新对象引用计数+1；在MRC中和release成对出现。

strong：在ARC中使用，告诉系统把这个对象保留在堆上，直到没有指针指向，并且ARC下不需要担心引用计数问题，系统会自动释放。

weak：在被强引用之前，尽可能的保留，不改变引用计数；weak引用是弱引用，你并没有持有它；它本质上是分配一个不被持有的属性，当引用者被销毁(dealloc)时，weak引用的指针会自动被置为nil。可以避免循环引用。

copy：一般用来修饰不可变类型属性字段，如：NSString、NSArray、NSDictionary等。用copy修饰可以防止本对象属性受外界影响，在NSMutableString赋值给NSString时，修改前者 会导致 后者的值跟着变化。还有block也经常使用 copy 修饰符，但是其实在ARC中编译器会自动对block进行copy操作，和strong的效果是一样的。但是在MRC中方法内部的block是在栈区，使用copy可以把它放到堆区。

readwrite：可以读、写；编译器会自动生成setter/getter方法。

readonly：只读；会告诉编译器不用自动生成setter方法。属性不能被赋值。

nonatomic：非原子性访问。用nonatomic意味着可以多线程访问变量，会导致读写线程不安全。但是会提高执行性能。

atomic：原子性访问。编译器会自动生成互斥锁，对 setter 和 getter 方法进行加锁来保证属性的 赋值和取值 原子性操作是线程安全的，但不包括可变属性的操作和访问。比如我们对数组进行操作，给数组添加对象或者移除对象，是不在atomic的负责范围之内的，所以给被atomic修饰的数组添加对象或者移除对象是没办法保证线程安全的。原子性访问的缺点是会消耗性能导致执行效率慢。

nonnull：设置属性或方法参数不能为空，专门用来修饰指针的，不能用于基本数据类型。

nullable：设置属性或方法参数可以为空。

null_resettable：设置属性，get方法不能返回为空，set方法可以赋值为空。

_Null_unspecified：设置属性或方法参数不确定是否为空。

后四个属性应该主要就是为了提高开发规范，提示使用的人应该传什么样的值，如果违反了对规范值的要求，就会有警告。

weak修饰的对象释放则自动被置为nil的实现原理：

Runtime维护了一个weak表，存储指向某个对象的所有weak指针。weak表其实是一个hash（哈希）表，Key是所指对象的地址，Value是weak指针的地址数组（这个地址的值是所指对象的地址）。

weak 的实现原理可以概括一下三步：

1、初始化时：runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。

2、添加引用时：objc_initWeak函数会调用 objc_storeWeak()函数， objc_storeWeak() 的作用是更新指针指向，创建对应的弱引用表。

3、释放时，调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。

### 13：成员变量ivar和属性property的区别，以及不同关键字的作用
- **成员变量：**成员变量的默认修饰符是@protected、不会自动生成set和get方法，需要手动实现、不能使用点语法调用，因为没有set和get方法，只能使用->。

- **属性：**属性会默认生成带下划线的成员变量和setter/getter方法、可以用点语法调用，实际调用的是set和get方法。

> 注意：分类中添加的属性是不会自动生成 setter/getter方法的，必须要手动添加。

> **实例变量：**class类进行实例化出来的对象为实例对象

关键字作用：

访问范围关键字
- @public：声明公共实例变量，在任何地方都能直接访问对象的成员变量。

- @private：声明私有实例变量，只能在当前类的对象方法中直接访问，子类要访问需要调用父类的get/set方法。

- @protected：可以在当前类及其子类对象方法中直接访问(系统默认)。

- @package：在同一个包下就可以直接访问，比如说在同一个框架。

关键字
- @property：声明属性，自动生成一个以下划线开头的成员变量_propertyName(默认用@private修饰)、属性setter、getter方法的声明、属性setter、getter方法的实现。**注意：**在协议@protocol中只会生成getter和setter方法的声明，所以不仅需要手动实现getter和setter方法还需要手动定义变量。

- @sythesize：修改@property自动生成的_propertyName成员变量名，@synthesize propertyName = newName；。

- @dynamic：告诉编译器：属性的 setter 与 getter 方法由用户自己实现，不自动生成。**谨慎使用：**如果对属性赋值取值可以编译成功，但运行会造成程序崩溃，这就是常说的动态绑定。

- @interface：声明类
- @implementation：类的实现
- @selecter：创建一个SEL，类成员指针
- @protocol：声明协议
- @autoreleasepool：ARC中的自动释放池
- @end：类结束

### 14:runloop
- runloop：通过系统内部维护的循环进行事件/消息管理的一个对象。runloop实际上就是一个do...while循环，有任务时开始，无任务时休眠。

- 其本质是通过mach_msg()函数接收、发送消息。

RunLoop 与线程的关系：

- RunLoop的作用就是来管理线程的，当线程的RunLoop开启后，线程就会在执行完任务后，处于休眠状态，随时等待接受新的任务，不会退出。
- 只有主线程的RunLoop是默认开启的，其他线程的RunLoop需要手动开启。所以当程序开启后，主线程会一直运行，不会退出。
runloop 事件循环机制内部流程

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19a31c9db7a04aa8ad7799a85a081779~tplv-k3u1fbpfcp-watermark.image)

RunLoop主要涉及五个类：

- CFRunLoop：RunLoop对象、

- CFRunLoopMode：五种RunLoop运行模式、

- CFRunLoopSource：输入源/事件源，包括Source0 和 Source1

- CFRunLoopTimer：定时源，就是NSTimer、

- CFRunLoopObserver：观察者，用来监听RunLoop。

CFRunLoop：RunLoop对象

CFRunLoopMode：RunLoop运行模式，有五种：
1. kCFRunLoopDefaultMode：默认的运行模式，通常主线程是在这个 Mode 下运行的。
2. UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。
3. UIInitializationRunLoopMode：在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。
4. GSEventReceiveRunLoopMode：接受系统事件的内部 Mode，通常用不到。
5. kCFRunLoopCommonModes：是一个伪模式，可以在标记为CommonModes的模式下运行，RunLoop会自动将

_commonModeItems里的 Source、Observer、Timer 同步到具有此标记的Mode里。

CFRunLoopSource：输入源/事件源，包括Source0 和 Source1两种：

1. Source1：基于mach_Port，处理来自系统内核或其它进程的事件，比如点击手机屏幕。
2. Source0 ：非基于Port的处理事件，也就是应用层事件，需要手动标记为待处理和手动唤醒RunLoop。

简单举例：一个APP在前台静止，用户点击APP界面，屏幕表面的事件会先包装成Event告诉source1(mach_port)，source1唤醒RunLoop将事件Event分发给source0，由source0来处理。

CFRunLoopTimer：定时源，就是NSTimer。在预设的时间点唤醒RunLoop执行回调。因为它是基于RunLoop的，因此它不是实时的（就是NSTimer 是不准确的。 因为RunLoop只负责分发源的消息。如果线程当前正在处理繁重的任务，就有可能导致Timer本次延时，或者少执行一次）。

CFRunLoopObserver：观察者，用来监听以下时间点：CFRunLoopActivity

- kCFRunLoopEntry：RunLoop准备启动

- kCFRunLoopBeforeTimers：RunLoop将要处理一些Timer相关事件

- kCFRunLoopBeforeSources：RunLoop将要处理一些Source事件

- kCFRunLoopBeforeWaiting：RunLoop将要进行休眠状态,即将由用户态切换到内核态

- kCFRunLoopAfterWaiting：RunLoop被唤醒，即从内核态切换到用户态后

- kCFRunLoopExit：RunLoop退出

- kCFRunLoopAllActivities：监听所有状态

各数据结构之间的联系：

1：Runloop和线程是一对一的关系

2：Runloop和RunloopMode是一对多的关系

3：RunloopMode和RunloopSource是一对多的关系

4：RunloopMode和RunloopTimer是一对多的关系

5：RunloopMode和RunloopObserver是一对多的关系

**为什么 main 函数能够保持一直存在且不退出？**
> 在 main 函数内部会调用 UIApplicationMain 这样一个函数，而在UIApplicationMain内部会启动主线程的 runloop，可以做到有消息处理时，能够迅速从内核态到用户态的切换，立刻唤醒处理，而没有消息处理时通过用户态到内核态的切换进入等待状态，避免资源占用。因此 main 函数能够一直存在且不退出。

### 15:runtime
#### 什么是runtime？

runtime 一套c、c++、汇编编写的API，为OC提供运行时功能。能够将数据类型的确定由编译期推迟到运行时。

问1：方法的本质，问2：runtime的消息机制

 **方法的本质其实就是发送消息。**
 **发送消息主要流程：**
* 快速查找：objc_msgSend查找cache_t缓存消息
* 慢速查找：递归自己和父类查找方法lookUpImpOrForward
* 查找不到消息，进行动态方法解析：resolveInstanceMethod
* 消息快速转发：forwardingTargetForSelector
* 消息慢速转发：消息签名methodSignatureForSelector和分发forwardInvocation
* 最终仍未找到消息：程序crash，报经典错误信息unrecognized selector sent to instance xxx

SEL是什么？IMP是什么？两者有什么联系？
- SEL是方法编号，即方法名称，在dyld加载镜像时，通过read_image方法加载到内存的表中了。
- IMP是函数实现指针，找IMP就是找函数的过程

两者的关系：sel相当于书本的目录标题，imp就是书本的页码。查找具体的函数就是想看这本书里面具体篇章的内容：

1). 我们首先知道想看什么，也就是title -sel

2). 然后根据目录对应的页码 -imp

3). 打开具体的内容 -方法的具体实现

**runtime应用：**

1.方法的交换：具体应用拦截系统自带的方法调用（Method Swizzling黑魔法）

2.实现给分类增加属性

3.实现字典的模型和自动转换

4.JSPatch替换已有的OC方法实行等

5.aspect 切面编程

#### 能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？

1.不能向编译后得到的类中增加实例变量。
2.可以向运行时创建的类中添加实例变量。
3.因为编译后的类已经注册在 runtime 中，类结构体中的 objc_ivar_list实例变量的链表和 instance_size 实例变量的内存大小已经确定，同时runtime会调用 class_setIvarLayout或class_setWeakIvarLayout来处理 strong、weak引用，所以不能向存在的类中添加实例变量。
运行时创建的类是可以添加实例变量，调用class_addIvar函数。但是得在调用objc_allocateClassPair之后，objc_registerClassPair之前，原因同上。

### 16:Category中添加属性和成员变量的区别

Category它的主要作用是在不改变原有类的前提下，动态地给这个类添加一些方法。
分类的结构体指针中，没有属性列表，只有方法列表。原则上它只能添加方法，不能添加属性(成员变量)，但是可以借助运行时关联对象 `objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);`、`objc_getAssociatedObject(self,@selector(name));。`
- 分类中的可以写@property，但不会生成setter/getter方法声明和实现，也不会生成私有的成员变量，会编译通过，但是引用变量会报错。
- 如果分类中有和原有类同名的方法，会优先调用分类中的方法，就是说会忽略原有类的方法，同名方法调用的优先级为 分类 本类 父类，因为方法是放在方法栈中，遵循先进后出原则；

### 17:isa指针

- isa是一个Class类型的指针，其源码结构为isa_t联合体，在类中以Class对象存在，指向类的地址，大小为8字节(64位)。
- 每个实例对象都有isa的指针指向对象的类。Class里也有个isa的指针指向meteClass(元类)。元类保存了类方法的列表。当类方法被调用时，先会从本身查找类方法的实现，如果没有，元类会向他父类查找该方法。元类（meteClass）也是类，它也是对象，也有isa指针。
- isa的指向：对象的isa指向类，类的isa指向元类(meta class)，元类isa指向根元类，根元类的isa指向本身，形成了一个封闭的内循环。isa可以帮助一个对象找到它的方法。
- isa指向图中类的继承关系：Teacher -> Person -> NSObject -> nil。这里需要注意的是根元类的父类是 NSObject，NSObject的父类是nil。

### 18:block
 **什么是Block Block是将函数及其执行上下文封装起来的对象。**

**什么是Block调用 Block调用即是函数的调用。**

#### Block的几种形式(类型)

**Block有三种形式，包括：**

* 全局Block(_NSConcreteGlobalBlock)：当我们声明一个block时，如果这个block没有捕获外部的变量，那么这个block就位于全局区(已初始化数据(.data)区)。

* 栈Block(_NSConcreteStackBlock)：

 * 1). ARC环境下，当我们声明并且定义了一个block，系统默认使用__strong修饰符，如果该Block捕获了外部变量，实质上是从__NSStackBlock__转变到__NSMallocBlock__的过程，只不过是系统帮我们完成了copy操作，将栈区的block迁移到堆区，延长了Block的生命周期。对于栈block而言，变量作用域结束，空间被回收。

 * 2). ARC的环境下，如果我们在声明一个block的时候，使用了__weak或者__unsafe__unretained的修饰符，那么系统就不会做copy的操作，也就不会将其迁移到堆区。

* 堆Block(_NSConcreteMallocBlock)：

 * 1). 在MRC环境下，我们需要手动调用copy方法才可以将block迁移到堆区

 * 2). 而在ARC环境下，__strong修饰的（默认）block捕获了外部变量就会位于堆区，NSMallocBlock支持retain、release，会对其引用计数＋1或 -1。

只有局部变量 - 和定义的属性 才会拷贝到堆区

* 1). 存储在程序的数据区域，在 block 内部没有引用任何外部变量。

* 2). 使用外部变量并且未进行copy操作的block是栈block。

* 3). 对栈block进行copy操作，就是堆block。对堆Block进行copy，将会增加引用计数。对全局block进行copy，仍是全局block。

在什么场景下使用__block修饰符呢？

* 1). 对截获变量进行赋值操作需要添加__block修饰符（赋值 != 使用）。

* 2). 对局部变量（基本数据类型和对象类型）进行赋值需要__block修饰符。 其内部其实是对该__block对象进行拷贝，所以通过__block可以修改被截获变量的值且不会和外部变量互相影响。

* 3). 对静态局部变量、全局变量、静态全局变量不需要__block修饰符。

#### block 的截获变量特性？

* 基本数据类型的局部变量 Block 可以截获其值。
* 对于对象类型的局部变量连同所有权修饰符一起截获。
* 局部静态变量以指针的形式进行截获。
* 全局变量和静态全局变量，block 是不截获的。

#### weak打破Block循环引用原理

- block内部操作的是weakSelf的指针地址，它和self是两个不同的指针地址，即 没有直接持有self，所以可以weakSelf可以打破self的循环引用关系 self -block - weakSelf。

