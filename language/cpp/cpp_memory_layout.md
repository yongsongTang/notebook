# cpp内存布局

## cpp对象内存布局

一个对象在内存中排列方式

### 对齐规则

**内存对齐：** 数据在内存中的存储地址必须是某个值的整数倍。以提高CPU访问内存的效率。比如，cpu一次性读取4字节，一个int类型的数据如果存储在`0x00-0x03`的位置一次读完了，如果存储在`0x01-0x04`位置，需要读取两次，一次读`0x00-0x03`，一次读`0x04-0x07`然后组合`0x01-0x04`的数据。

#### 基本数据类型

| 类型   | 长度                | 对齐要求                |
| ------ | ------------------- | ----------------------- |
| char   | 1字节               | 1字节                   |
| short  | 2字节               | 2字节                   |
| int    | 4字节               | 4字节                   |
| float  | 4字节               | 4字节                   |
| long   | 4(32位)/8(64位)字节 | 4(32位)/(64位)字节对齐  |
| double | 8字节               | 8字节对齐               |
| void*  | 4(32位)/8(64位)字节 | 4(32位)/8(64位)字节对齐 |

char 占用1个字节，1字节对齐，那么char类型数据可以出现在任何内存地址处。int 占用4字节内存，4字节对齐，那么int类型的数据只能出现地址为4的整数倍位置处。

#### 结构体或者类

- 按照变量声明顺序依次类型对齐。
- 结构体或者类，总的大小是其成员变量中最大的对齐要求。

```cpp
class Base {
    short s_short = 99; //2字节，第0-1字节
    const char *s_pc = "8 or 4"; // 8字节(字节对齐)，第8-15字节，第2-7字节填充
    char s_c = 'a'; // 1字节
}; // 2 + [6] + 8 + 1 = 17 +[7] = 24 总大小安装成员变量中最大的对齐要求，8字节对齐，所以末尾填充7字节
```

Base对象内存排布方式，2字节(0-1)存储`short`变量，`char *`8字节对齐，只能出现在地址为8的位置，所以填充6字节(2-7)，第8-15字节存储`char *`变量，第16字节存储`char`。现在类总大小2+6+8+1=17字节，按照成员变量中最大的对齐方式(来自`char *`的8字节)对齐，所以填充7字节。总大小17+7=24字节

| 字节偏移 | 0-1     | 2-7  | 8-15 | 16  | 17-23 |
| -------- | ------- | ---- | ---- | --- | ----- |
| 变量     | s_short | 填充 | s_pc | s_c | 填充  |

```cpp
    printf("%lu\n", sizeof(Base)); // 24
    Base b;
    uint64_t base = (uint64_t) &b;
    // 第16字节位置 存储chard s_c变量
    base += 16;
    auto base_c = (char *) base;
    printf("c: %c\n", *base_c); // a
```

#### 类中包含方法

```cpp
class Base {
    short s_short = 99; //2字节
    const char *s_pc = "8 or 4"; // 64位机器8字节
    char s_c = 'a'; // 1字节

    // 静态变量，类的属性，所有对象共享 存储在数据段  ==> 不占用对象内存空间
    static int s_static;

    // 非虚方法，所有对象共享，存储在代码段 ==> 不占用对象内存空间
    void dump() { printf("Base %s %d %s %c\n", __func__, s_short, s_pc, s_c); }

    // 虚方法，对象开始位置增加一指针，指向虚函数表(vtable)。虚函数表是个函数指针数组
    // 虚函数表存储在只读数据段(或代码段取决于编译器)，每个虚函数在表中有一条记录
    virtual void virtual_f() { printf("Base %s\n", __func__); };
}; // 8 + 2 + [6] + 8 + 1 = 25 +[7] = 32
```

##### 非虚方法

非虚成员方法，所有对象共享的，只有一份存储在代码段。静态成员变量，类的属性，所有对共享，存储在数据段。此时对象占用内存大小并没有改变。

##### 虚方法

类中存在虚方法(包括虚析构函数)，对象内存的开始位置增加一指针，指向虚函数表(vtable)。虚函数表是个函数指针数组。此时对象开始位置增加了8字节(64位)存储执行虚函数表的指针。

```cpp
    printf("%zu \n", sizeof(Base));  // 32
    Base b;
    auto base = (uint64_t) &b;
    // 虚函数
    void ***p_vtable = (void ***) base;
    void **vtable = *p_vtable; // 虚函数表
    // 第0个虚函数 调用
    using Type_fun = void (*)();
    auto f = (Type_fun) vtable[0];
    f(); // 虚函数virtual_f方法被调用
    
    // 第16字节位置 存储char* s_pc变量
    base += 16;
    auto base_spc = (char **) base;
    printf("c: %s\n", *base_spc); // "8 or 4"
```

| 字节偏移  | 0-7 | 8-9  | 10-15  | 16-23 | 24  | 25-31 |
| -------- | ---- | ---- | ---- | --- | ---- | ---- |
| 变量     | p_vtable 指向vtable | s_short | 填充 | s_pc | s_c | 填充  |

第0-7字节存储指向虚函数表的指针。下一个s_short变量2字节对齐，从第8位置开始不用填充， 第8-9字节存储s_short。下一个s_pc变量8字节对齐，得从第16字节开始存储，所以填充6字节(10-15)，从第16字节开始后的8字节(16-23)存储s_pc。第24字节存储s_c。此时共25字节，按照类的最大成员变量对齐(8字节)，填充7字节(25-31)。

#### 类中有继承关系

```cpp
class Base {
public:
    short s_short = 99; //2字节
    const char *s_pc = "8 or 4"; // 64位机器8字节
    virtual void virtual_f() { printf("Base %s\n", __func__); }  // 8

    virtual void virtual_f1() { printf("Base %s\n", __func__); }  // 8
}; // 8(指向虚函数表的指针) + 2(s_short) + [6](填充) + 8(s_pc) = 24

class Child : private Base {
public:
    bool c_b = false;
    // 子类和基类 同名成员变量 ok
    short s_short = 212;

    void virtual_f() override { printf("child %s\n", __func__); }
};// 24(基类) + 1(c_b) + [1](填充) + 2(s_short) = 28 +[4](填充) = 32
```

Base基类内存布局

| 字节偏移  | 0-7 |  8-9  | 10-15  | 16-23 |
| -------- | ---- | ---- | ---- | ---- |
| 变量     | 指向虚函数表的指针 | s_short | 填充 | s_pc |

Child扩展类对象内存布局

| 字节偏移  | 0-23 | 24  | 25  | 26-27 | 28-31  |
| -------- | ---- | ---- | ---- | --- | ---- |
| 变量     | Base基类 | c_b | 填充 | s_short | 填充 |

每个有虚函数的*类*(Base，Child)有自己的虚函数表，所有对象共享虚函数表，调用类非静态方法时隐式传入指向类对象的this指针。没有复写基类虚方法时，子类和基类虚函数表内容相同(只是内容相同，子/基类有自己的虚函数表)，复写了某个虚方法时，修改对应虚函数表指向对应方法。

> - 不同编译器内存布局可能不一样
> - 虚函数表的前一个指针既`void **vtable = *(void ***) &c; auto *pTypeInfo = static_cast<std::type_info *>(vtable[-1]);
`，指向的是`std::type_info*`，如果有虚函数并且编译时没有禁用`-fno-rtti`的话。

```cpp
    // 关于虚函数表
    Bean b;
    auto address = (uint64_t) &b;
    auto pBean = (Bean *) address;

    // 虚函数表指针，一个指针 指向虚函数表
    auto p_vtable = (void ***) address;

    // 函数指针数组(虚函数表)，数组 数组每个元素是一个指向虚函数的指针
    void **vtable = *p_vtable;

    // 第一个虚函数指针
    void *v0 = *(vtable + 0); // vtable[0];

    // 第二个虚函数指针
    void *v1 = *(vtable + 1); // vtable[1];

    // 函数指针变量声明, 返回值(*p)(参数)
    // void (*p)(Bean *);

    // 函数指针, 第一个虚函数类型指针。 类方法第一个参数是this
    using V0_Type = void (*)(Bean *);
    auto ev0 = V0_Type(v0);
    ev0(nullptr); // 

    using V1_Type = void (*)(Bean *, int);
    auto ev1 = V1_Type(v1);
    (*ev1)(nullptr, 23);
```

同一个地址既可以转成`void ***`，又可以转成`Bean *`。地址就是一个数值，本身是没有类型的，人为加上了类型，方便访问。类型与实际类型不匹配的话崩溃。

- `auto pBean = (Bean *) address;`, address内存地址处，放着一个完整的`Bean`类型。可以通过Bean的访问方式，访问对应的内存空间。
- `auto p_vtable = (void ***) address;`，address内存地址处，放着`void **`类型数据(函数指针数组，虚函数表)。

```cpp
    // 指向类中成员方法、成员变量的指针  void (Bean::*pFunction)();  int Bean::* pMember;
    void global_fun() { }
    // 函数指针，指向 无参数、返回void的函数
    void (*pf)() = global_fun;
    
    // 函数指针，指向Bean类的，无参数、返回void的成员函数(任何来自Bean类的 无参数 返回void的成员函数)。
    void (Bean::*pFunction)();
    pFunction = &Bean::class_fun; // .text段, 类(Bean)中方法(class_fun)的地址，使用时候要加上this。地址值但是不允许转uint64_t

    // 使用，不同于普通函数指针，不能pFunction()使用。类成员方法，访问需要提供类指针(this)
    Bean *p = nullptr;
    (p->*pFunction)(); // pFunction方法中没有用到this， 所可以传nullptr

    /***********************************/

    // 指针，指向Bean类中的一个int类型的成员变量(任何来自Bean类的int类型的成员变量)。
    int Bean::* pMember;
    pMember = &Bean::class_val; // pMember，class_val成员相对于Bean对象起始地址的偏移值。虽然是个int值 但是不允许强转int

//  使用，不同于普通指针，不能*pMember使用。类成员变量，访问需要提供类指针(this)
    Bean b;
    int member = b.*pMember; // 访问类成员变量，必须有实质对象(b)
```

## 内存段

### 运行时内存段

运行时将虚拟内存按照逻辑功能分为一段一段的，以方便管理。比如：代码段(.text)存放编译后方法的二进制指令，只读防止代码注入。编译后的可执行文件，动态，静态库也是分段存储的(文件内存段)，在运行时按照分段加载到内存中。静态库在连接时，同名段合并到可执行文件。动态库则是在运行时独立映射(mmap)到内存中，所有进程共享，通过plt，got机制访问。

| 地址  | 段 |  名称  | 存储数据  | 权限 | 说明 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| 0x0  |    |       |          |    |  程序头，系统信息  |
|0x0000555555555020-0x0000555555555050|.plt | 过程连接表 | 每个被引用动态库方法生成的一小段代码 | 读、执行 |  |
|0x0000555555555060-0x00005555555551b6|.text| 代码段 | 方法(全局方法 类方法)编译后可执行的二进制指令 | 读、执行 | 共享库的代码段被多个进程共享 |
|0x0000555555556000-0x000055555555601e|.rodata| 只读数据| 常量()，虚函数表，类信息 | 读 | |
|0x0000555555557fb8-0x0000555555557fe8|.got|全局偏移表|保存动态库中全局变量的地址| 读、写| |
|0x0000555555557fe8-0x0000555555558010|.got.plt| | 指针方法数组，每个元素都是指针，指向动态库中方法| 读、写| |
|0x0000555555558010-0x0000555555558020|.data|数据段| 初始化的全局变量，静态变量(包括类静态变量) | 读、写| |
|0x0000555555558020-0x0000555555558028|.bss| | 未初始化的全局变量，静态变量(包括类静态变量) |读、写| 加载时初始化为0，代码初始化全0的，编译器可能优化到bss段 |
|0x00005d537722b000-0x00005d53775f6000|.heap| 堆 | 动态分配的内存，手动管理 | 读、写 | 可能产生内存碎片 |
|0x00007ffff7fc52a8-0x00007ffff7a11d90|.mmp|映射|动态库，文件，匿名映射| 多个权限 | 每个共享库独立分段，映射到这一区域 |
|0x00007ffc11ad4000-0x00007ffc11afc000|.stack|栈 | 线程栈，局部变量，自动管理 | 读、写 |  |

> 中间缺了很多分段，真实分段更多更详细。可以在gdb下用`info file`查看运行时各个段及其地址。或者`cat /proc/{pid}/maps`看运行时地址。

#### 代码段

存储编译后方法的二进制执行。在编译后方法的地址就确定了，有两种情况，相对地址(位置无关)和绝对地址(位置无关)。

1. 位置有关代码，编译后方法地址，就是运行时方法量地址，代码段必须加载到指定内存位置。
2. 位置无关代码(PIC)，编译后方法地址是基于代码段地址的偏移量。代码段可以加载到任意内存位置。

> - 位置无关代码(PIC)，运行时相对地址计算，所以比绝对地址，直接调用要慢一点(忽略不计)，但是可以防止内存攻击。默认编译为位置无关代码(PIC)。
> - 动态库是运行时映射到内存中，编译时还不知道地址。

调用非虚方法时，通过绝对地址或者相对地址(rip寄存器(pc寄存器)+偏移量)，找到对应方法。调用虚方法时，先访问对象的虚函数表，在通过虚函数表找到对应方法。

#### data段

存储初始化的全局变量，静态变量。记录变量，类型，初始值。初始值可能非常大，比如：`int arr[1024 * 1024 * 10] = {9, 4, 2};` 40M，意味着数据段只少要存储40M的数据，可执文件大小40M起。*影响可执行文件大小*

```shell
# 初始化的全局变量 int arr[1024 * 1024 * 10] = {9, 4, 2 };
➜ size main_seg                    
   text    data     bss     dec     hex filename
    173 41943048              0 41943221        28000b5 main_seg

➜ ls -lh main_seg # 应为要保存初始值，可执行文件增大
-rwxr-xr-x@ 1 tys  staff    40M Nov 25 19:34 main_seg
```

#### bss段

存储未初始化的全局变量，静态变量。只是记录所需的内存大小，所以不影响可执行文件的大小。运行时，分配这么大一块内存(程序bss段)，然后清零。*只是记录.bss所需的大小，不影响可执行文件大小*

```shell
# 未初始化的全局变量 int arr[1024 * 1024 * 10]; 不占用文件大小
➜ size main_seg                    
   text    data     bss     dec     hex filename
    173       8 41943040        41943221        28000b5 main_seg

➜ ls -lh main_seg
-rwxr-xr-x@ 1 tys  staff   8.3K Nov 25 19:43 main_seg

# objdump -d main
0000000000001150 <main>:
    # 未初始化的全局变量，在.bss段存储的位置是确定的 通过rip(指令指针寄存器寻找)
    115f:       8b 05 cb 2e 00 00       mov    0x2ecb(%rip),%eax   # 4030 <global_uninit_Val>
```

- 编译时可计算出值(非0)的变量在`.data`段，否则在`.bss`段
- 类类型变量，编译时每个成员变量值都确定，在`.data`段，否则`.bss`段。如果对象创建依赖自定义构造函数，即使每个成员变量值确定，也在`.bss`段。
- 初始值为0的变量(类，成员变量全为0)，编译器优化放在`.bss`段
  
#### 堆

动态分配的内存，手动管理。所有线程共享(除线程本地变量 tls变量) `new delete`，`malloc free`，分配释放不同大小内存，可能造成内存碎片化。

#### 映射区

内存映射(mmap)，动态库，文件，匿名映射。多个进程共享同一个物理页，写时复制。每个动态库映射到自己独立的内存空间，并且每次映射的位置是不确定的。

#### 栈

[线程栈](#线程栈)，自动管理。每个线程自己独立的栈空间(8M)，生命周期固定，过深的递归调用造成栈溢出。存储返回地址(call 指令自动压入)，方法参数，上个栈帧基地址(方法执行结束，返回到调用的地方，还有这个方法栈帧基地址)，局部变量。

```shell
# 栈空间大小查看
➜ ulimit -s
8192
```

**堆，栈对比：**

- 生命周期：栈，随线程的创建而创建，线程退出而结束。堆，需要手动创建和回收。
- 作用范围：栈，每个线程独立的。堆，线程之间共享，多线程之间竞争需要加锁。
- 大小：栈，单个栈大小固定，每个线程都预留8M虚拟空间，所有加起来才是总栈大小。堆，非常大，和虚拟地址空间大小相关。
- 内存创建回收：栈，栈中内存空间的创建和回收只是栈指针的移动，速度非常快。堆，堆中空间的创建回收，涉及系统调用内存管理(查找合适大小的内存空间，碎片的整理合并)。
- 读写速度：栈，内存分配通常连续的，更容易被cpu缓存命中。堆，访问是不确定的，可能导致更多缓存未命中。

> cpu缓存命中：cpu并不是每次从内存中读取它需要的字节数，而是按缓存行大小读取(64k)。如果要访问的数据在缓存行中，就不需要从内存中在读取数据。cpu缓存命中比从内存中加载快得多。缓存行是CPU与内存之间数据传输的基本单位。

#### 对象创建

在堆(heap)，栈(stack)上，分配对象所需的内存大小。并且按照指定值或者默认值设置这块内存区域。

> `Bean a`栈上0x7fffffffd0c8到0x7fffffffd0d0内存区域(8字节)存储`Bean`对象的成员变量。指针就是，内存中存储这个对象开始地址0x7fffffffd0c8。后续通过这个首地址加上偏移，来访问实例中成员变量。编译结束栈上内存分配已经规划好了，`lea -0x8(%rbp),%rdi`,`call 1210 <_ZN7BeanClsC2Ev>` 栈基地址 - 0x8字节 = 存储对象实例开始地址处。

##### 默认初始化 T

`BeanCls bean;`,`auto *p = new BeanCls;`

1. 分配内存(this)，栈：rsp指针移动  堆：operator new(size_t)
2. 无参构造函数。成员变量的值，是定义时默认值。没有默认值的成员变量，不做任何操作随机值。

```shell
# BeanCls bean;
0000000000001150 <_Z18fun_def_init_stackv>:
    1150:       55                      push   %rbp
    1151:       48 89 e5                mov    %rsp,%rbp
    1154:       48 83 ec 10             sub    $0x10,%rsp
    1158:       48 8d 7d f8             lea    -0x8(%rbp),%rdi # 先分配内存空间
    115c:       e8 8f 00 00 00          call   11f0 <_ZN7BeanClsC2Ev> # 在调用构造方法，给分配的空间设置值
    ...

#auto *p = new BeanCls;
0000000000001170 <_Z17fun_def_init_heapv>:
    1170:       55                      push   %rbp
    1171:       48 89 e5                mov    %rsp,%rbp
    1174:       48 83 ec 20             sub    $0x20,%rsp
    1178:       bf 08 00 00 00          mov    $0x8,%edi
    117d:       e8 be fe ff ff          call   1040 <_Znwm@plt> # operator new(size_t) 堆上分配
    1182:       48 89 c7                mov    %rax,%rdi # rax 堆上分配的内存首地址(this)。rdi 构造函数参数
    1185:       48 89 7d e8             mov    %rdi,-0x18(%rbp) # auto *p = ，this指针存在栈上 
    1189:       e8 62 00 00 00          call   11f0 <_ZN7BeanClsC2Ev> # 构造函数

    118e:       48 8b 45 e8             mov    -0x18(%rbp),%rax
    1192:       48 89 45 f8             mov    %rax,-0x8(%rbp)
    1196:       48 8b 45 f8             mov    -0x8(%rbp),%rax # 冗余指令？？ O0

    119a:       48 89 45 f0             mov    %rax,-0x10(%rbp)
    119e:       48 83 f8 00             cmp    $0x0,%rax # 堆上分配地址是否是0
    11a2:       0f 84 09 00 00 00       je     11b1 <_Z17fun_def_init_heapv+0x41> # 等于0，分配堆上分配失败。跳过delete 
    11a8:       48 8b 7d f0             mov    -0x10(%rbp),%rdi # 不等于0
    11ac:       e8 7f fe ff ff          call   1030 <_ZdlPv@plt> # delete p
    11b1:       48 83 c4 20             add    $0x20,%rsp # 栈上内存回收
    ...
```

##### 值初始化 T()

`auto *p = new BeanCls();`

1. 分配空间，operator new(size_t)
2. 2.1 没有定义构造函数，0初始化(memset) + 构造函数
   2.2 定义构造函数，构造函数

```shell
00000000000011e0 <_Z17fun_val_init_heapv>:
    11e0:       55                      push   %rbp
    11e1:       48 89 e5                mov    %rsp,%rbp
    11e4:       48 83 ec 20             sub    $0x20,%rsp
    11e8:       bf 08 00 00 00          mov    $0x8,%edi
    11ed:       e8 5e fe ff ff          call   1050 <_Znwm@plt> # operator new(size_t)
    11f2:       48 89 c7                mov    %rax,%rdi
    11f5:       48 89 7d e8             mov    %rdi,-0x18(%rbp)
    11f9:       31 f6                   xor    %esi,%esi # x or x = 0
    11fb:       ba 08 00 00 00          mov    $0x8,%edx
    1200:       e8 2b fe ff ff          call   1030 <memset@plt> # 0初始化  memset(this,0,0x8), 定义了构造函数 就没有0初始化了
    1205:       48 8b 7d e8             mov    -0x18(%rbp),%rdi
    1209:       e8 52 00 00 00          call   1260 <_ZN7BeanClsC2Ev> # 构造函数
    120e:       48 8b 45 e8             mov    -0x18(%rbp),%rax
    1212:       48 89 45 f8             mov    %rax,-0x8(%rbp)
    1216:       48 8b 45 f8             mov    -0x8(%rbp),%rax
    121a:       48 89 45 f0             mov    %rax,-0x10(%rbp)
    121e:       48 83 f8 00             cmp    $0x0,%rax
    1222:       0f 84 09 00 00 00       je     1231 <_Z17fun_val_init_heapv+0x51>
    1228:       48 8b 7d f0             mov    -0x10(%rbp),%rdi
    122c:       e8 0f fe ff ff          call   1040 <_ZdlPv@plt>
    1231:       48 83 c4 20             add    $0x20,%rsp
    ...
```

>`()` 只要定义了构造函数，即使是无参构造函数，就没有0初始化。

##### 列表初始化 T{}

`BeanCls a{}`,`new BeanCls{}`

1. 分配空间，栈：rsp指针移动  堆：operator new(size_t)
2. 成员变量。优先级 指定值 > 默认值 > 0

- `BeanCls a{}`,`new BeanCls{}` 要求有无参构造函数。
- `BeanCls a{1}`,`new BeanCls{2}` 声明顺序初始化 或者 对应构造函数
- `BeanCls a{.class_val=0x1}`,`new BeanCls{.class_val=0x1}` 指定初始化, 要求没有定义构造函数。

```shell
# BeanCls a{}
0000000000001240 <_Z21fun_setval_init_stackv>:
    1240:       55                      push   %rbp
    1241:       48 89 e5                mov    %rsp,%rbp # 直接赋值，没有调用构造函数
    1244:       c7 45 f8 5a 00 00 00    movl   $0x5a,-0x8(%rbp) # 有默认值的成员变量
    124b:       c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%rbp) # 没有默认值的成员变量，设置0
    1252:       5d                      pop    %rbp
    1253:       c3                      ret
# new BeanCls{}
0000000000001260 <_Z20fun_setval_init_heapv>:
    1260:       55                      push   %rbp
    1261:       48 89 e5                mov    %rsp,%rbp
    1264:       48 83 ec 10             sub    $0x10,%rsp
    1268:       bf 08 00 00 00          mov    $0x8,%edi
    126d:       e8 de fd ff ff          call   1050 <_Znwm@plt> # operator new(size_t)
    1272:       c7 00 5a 00 00 00       movl   $0x5a,(%rax) # 直接赋值，没有调用构造函数
    1278:       c7 40 04 00 00 00 00    movl   $0x0,0x4(%rax) # 没有默认值的成员变量，设置0
    127f:       48 89 45 f8             mov    %rax,-0x8(%rbp)
    1283:       48 8b 45 f8             mov    -0x8(%rbp),%rax
    1287:       48 89 45 f0             mov    %rax,-0x10(%rbp)
    128b:       48 83 f8 00             cmp    $0x0,%rax
    128f:       0f 84 09 00 00 00       je     129e <_Z20fun_setval_init_heapv+0x3e>
    1295:       48 8b 7d f0             mov    -0x10(%rbp),%rdi
    1299:       e8 a2 fd ff ff          call   1040 <_ZdlPv@plt>
    129e:       48 83 c4 10             add    $0x10,%rsp
    ...
```

> `{}`，只要触发带参数构造函数`(BeanCls a{1}`,`new BeanCls{2})`，成员变量初始值权限交给构造函数。没有默认值的成员变量，不做任何操作随机值。

## 编译过程

源文件和库文件生成可执行文件的过程

### 预处理 preprocess

`clang++ -E main_test_compile.cpp -o main_test_compile.i`，预处理 = 头文件展开(多层include 递归展开)，宏替换

```cpp
// main_test_compile.i
# 9 "./utils.h" 2

namespace Utils {  
    float getId(float sid);
}

// 宏变量 float pi = M_PIf; 
float pi = 3.14159265358979323846f;

// 宏函数 int error = errno;
int error = (*__errno_location ());
```

### 编译 compile

`clang++ -S main_test_compile.cpp -o main_test_compile.s`，预处理 + 编译 = 汇编代码

```shell
# main 方法
main:                                   # @main
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $16, %rsp  # 栈分配空间
    movl    $0, -4(%rbp)
    leaq    .L.str.1(%rip), %rdi # .L.str.1 标签地址存(字符串) rdi   printf参数
    movb    $0, %al # printf参数
    callq   printf@PLT # printf("===start===\n");
    movss   .LCPI1_0(%rip), %xmm0           # xmm0 = [3.14159274E+0,0.0E+0,0.0E+0,0.0E+0]
    movss   %xmm0, -8(%rbp) # 栈上一块存储xmm0，也就是本地变量保存xmm0值  float pi = M_PIf;
    callq   __errno_location@PLT # 动态库中方法
    callq   _ZN5Utils5getIdEf@PLT # 其他目标文件中方法
    callq   _ZN4Bean14cls_static_funEv # 类静态方法
    addq    $16, %rsp # 栈内存回收
    popq    %rbp
```

### 汇编 assemble

`clang++ -c main_test_compile.cpp -o main_test_compile.o`，预处理 + 编译 + 汇编 = 单个文件机器码
为单个文件(.cpp)生成节头，符号表，重定位表

#### 节头(Section Headers)

描述目标文件或者可执行文件，各个节(段)的信息。包括，节(段)开始的地址，占用字节数，索引编号。

链接时读取多个文件(.o文件和库文件)的节头，合并节(同名的节合并成一个更大的节)。

目标文件(.o文件)中还没有链接，最终地址不确定，所以此时开始地址是0。索引编号和符号表对照看，确定符合属于那一节(段)。

```shell
# 查看目标文件节头
➜ readelf main_test_compile.o -S                         
There are 22 section headers, starting at offset 0xa70:

节头：
 [号](索引编号) 名称              类型           地址(未链接地址还不知道)   偏移量
       大小              全体大小          旗标   链接   信息   对齐
    ...
  [ 2] .text             PROGBITS         0000000000000000  00000040
       000000000000010c  0000000000000000  AX       0     0     16
  [ 3] .rela.text        RELA             0000000000000000  000005b0
       00000000000001b0  0000000000000018   I      21     2     8
  [ 4] .rodata.cst4      PROGBITS         0000000000000000  0000014c
       0000000000000004  0000000000000004  AM       0     0     4
  [ 7] .text._ZN5Ut[...] PROGBITS         0000000000000000  00000170  # 模板方法，单独生产了一个节（不同参数调用2次）
       0000000000000010  0000000000000000 AXG       0     0     16
  [ 9] .text._ZN5Ut[...] PROGBITS         0000000000000000  00000180  # 模板方法，单独生产了一个节（不同参数调用2次）
       0000000000000012  0000000000000000 AXG       0     0     16
  [10] .bss              NOBITS           0000000000000000  00000194
       0000000000000008  0000000000000000  WA       0     0     4
  [11] .data             PROGBITS         0000000000000000  00000194
       0000000000000008  0000000000000000  WA       0     0     4
        ...
  [21] .symtab           SYMTAB           0000000000000000  000002c8
       00000000000002e8  0000000000000018           1    12     8
```

#### 符号表(.symtab)

*符号：* 可读的一个名字，代表着一个方法或者变量在内存中的地址。C++存在命名空间，方法重载(出现同名方法)，名字会被混淆。比如：`_Z12test_lib_addii`，`_ZN4Bean7cls_funEv`。
方法(非静态全局方法，类方法)，变量(非静态全局变量，静态类成员变量，静态局部变量)生成符号。局部变量，const常量，不生符号，编译成代码后初始值是确定(。值或者0或者不赋值)。有些是调用了才生存符号，比如：模版方法`template<class T> void template_fun(const T &a) {}`。在调用模板方法的cpp文件中生成`WEAK`类符号。

> - `static`作用于全局方法和变量时，表示该符号仅对当前cpp可见。只有在内部有调用情况下生成`LOCAL`类符号。
> - `static`作用于局部变量时，表示该符号只初始化一次，并且生命周期延长至整个进程生命周期。(生命周期和全局符号一样，只对局部可见)。
> - `static`作用于类成员时，表示该符号属于类属性，所有对象共享相同属性。(生命周期和全局符号一样)。

符号表，记录符号对应内容存储在那个地址(这个地址是相对于它所在节(Ndx)的偏移)。符号表告诉链接器，当前编译单元对外提供哪些符号，使用了哪些符号，其中有些符号不认识，把他记录在了重定位符号表。

> 从符号表可以看出定义了哪些符号(有地址，类型)，哪些符号不认识(没有地址，类型，UND)。符号表对于运行并不是必须，可通过`strip`命令删掉以减小体积。

```shell
# 目标文件(.o)符号表
➜ readelf main_test_compile.o -s 

Symbol table '.symtab' contains 32 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     3: 0000000000000000     4 OBJECT  LOCAL  DEFAULT   12 .L.str # 
     4: 0000000000000004    11 OBJECT  LOCAL  DEFAULT   12 .L__func__._Z10g[...]
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 .text._ZN5Utils1[...] # 模板方法 生成了独立的段
    10: 0000000000000000     0 SECTION LOCAL  DEFAULT    9 .text._ZN5Utils1[...] # 模板方法 生成了独立的段
    11: 0000000000000004     4 OBJECT  LOCAL  DEFAULT   11 _ZZ10global_funv[...] # 全局方法中的静态变量 static int flag = true;
    12: 0000000000000000    27 FUNC    GLOBAL DEFAULT    2 _Z10global_funv # 全局方法
    13: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
    14: 0000000000000020   269 FUNC    GLOBAL DEFAULT    2 main
    16: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZN5Utils5getIdEf  # 命令空间下全局方法 独立文件定义实现(Utils.h Utils.cpp)
    17: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZN4Bean14cls_st[...] # 类静态方法  独立文件定义实现(Bean.h Bean.cpp)
    18: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZN4BeanC1Ev # 类构造方法
    19: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZN4Bean7cls_funEv  # 类非静态成员方法
    20: 0000000000000000    16 FUNC    WEAK   DEFAULT    7 _ZN5Utils12templ[...] # 模板方法 Utils::template_fun(1);  WEAK 若符号
    21: 0000000000000000    18 FUNC    WEAK   DEFAULT    9 _ZN5Utils12templ[...] # 模板方法 Utils::template_fun(1.0); WEAK 若符号
    23: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _Z12test_lib_addii # 动态库中方法
    24: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND gloab_lib # 动态库中变量
    25: 0000000000000004     4 OBJECT  GLOBAL DEFAULT   10 global_uninit_val #未初始化全局变量 int global_uninit_val;
    26: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _ZN4BeanD1Ev # 类析构方法
    27: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _Unwind_Resume
    28: 0000000000000000     1 OBJECT  GLOBAL DEFAULT   10 _ZN5Utils5flagXE  #命名空间下全局变量 bool flagX = false
    29: 0000000000000000     4 OBJECT  GLOBAL DEFAULT   11 global_val # 全局变量 int global_val = 321;
```

*Name:* 符号名称
*Ndx:* 对应[节头](#节头section-headers)中的索引编号，`UND`未定义，表示来自其他目标文件(.o)或者库。比如：`10 global_uninit_val`, 10在节头表中，对应`.bss`段所以`global_uninit_val`符号属于.bss段。
*Bind:* `LOCAL`，符号只在当前文件内部可见，对其他文件是隐藏的。`GLOBAL`，符合全局可见，可以被其他文件引用。`WEAK`，符号全局可见，但是可以被覆盖。同一符号可以既有一个`GLOBAL`引用，也有一个`WEAK`引用，链接时会优先选择使用`GLOBAL`的符号。也可以有多个同名的`WEAK`符号，链接时随机选择一个，比如：同一个模版方法在多个cpp文件中调用，。

> 通过在变量或者方法前添加`__attribute__((weak))`手动创建`WEAK`符号，`__attribute__((weak)) void weak_fun(int a)`，`__attribute__((weak)) int weak_var;`。链接时如果有对应符号的`GLOBAL`引用，则使用`GLOBAL`符号对应的变量或者方法，否则使用使用`WEAK`符号对应的变量或者方法。提供了一种编译时声明，链接时选择的扩展方式。

*Type：* SECTION，节(段)。OBJECT，变量。FUNC，方法。NOTYPE，没有类型(符号来自其他目标文件(.cpp)或者库)。TLS，线程本地变量。
*Size：* 符号内容占用的字节数
*Value：* 符号内容地址相对于所在段(Ndx)的偏移。比如：`0000000000000004     4 OBJECT  LOCAL  DEFAULT   11 _ZZ10global_funv[...]`，_ZZ10global_funv符号对应的内容存储在，.data段(11)偏移4字节地址处。对于来自于其他目标文件(.cpp)和库中的UND符号，Value 字段没有节内偏移量，编译时他不属于该文件任何一节，他的值为0。在重定位符号表中会存在一条UND符号的记录，地址计算转移到了重定位符号表。

```shell
# .data 段内容
➜ objdump -s -j .data main_test_compile.o

main_test_compile.o：     文件格式 elf64-x86-64

Contents of section .data:
 0000 41010000 01000000                    A.......     
```

`_ZZ10global_funv`符号对应值，.data段偏移4字节，0x00000001，代码是`static int flag = true;`。 `global_val`符号对应值，.data段偏移0字节，0x00000141(十进制321)，代码是`int global_val = 321`。通过符号表中描述的符号地址，找到符号对应的值，和代码中的值吻合。

#### 重定位表(.rela)

记录在哪个地址，引用了哪个符号，这个地址需要修正。告诉链接器，当前阶段不知道引用的符号在那个位置，当你知道符号最终位置后，把他填在需要修正的地址处。

```shell
# 目标文件(.o)的重定位表  链接前
➜ readelf main_test_compile.o -r
# .text段 重定位表
重定位节 '.rela.text' at offset 0x5e8 contains 21 entries:
  偏移量          信息           类型           符号值        符号名称 + 加数
000000000007  000300000002 R_X86_64_PC32     0000000000000000 .L.str - 4 #  "%s\n"， 常量字符串 
00000000000e  000400000002 R_X86_64_PC32     0000000000000004 .L__func__._Z10gl[...] - 4     # __func__
000000000015  000d00000004 R_X86_64_PLT32    0000000000000000 printf - 4    #  printf("%s\n", __func__);
000000000057  001000000004 R_X86_64_PLT32    0000000000000000 _ZN5Utils5getIdEf - 4 # 其他目标文件中方法
000000000061  001100000004 R_X86_64_PLT32    0000000000000000 _ZN4Bean14cls_sta[...] - 4 # 类静态方法
00000000006e  001200000004 R_X86_64_PLT32    0000000000000000 _ZN4BeanC1Ev - 4 # Bean 构造方法
00000000007d  001300000004 R_X86_64_PLT32    0000000000000000 _ZN4Bean7cls_funEv - 4 # 类方法
000000000092  001400000004 R_X86_64_PLT32    0000000000000000 _ZN5Utils12templa[...] - 4 # 模板方法 同一模板，不同参数
0000000000ae  001500000004 R_X86_64_PLT32    0000000000000000 _ZN5Utils12templa[...] - 4 # 模板方法 同一模板，不同参数
0000000000cc  001700000004 R_X86_64_PLT32    0000000000000000 _Z12test_lib_addii - 4 # 动态库中方法
0000000000d8  00180000002a R_X86_64_REX_GOTP 0000000000000000 gloab_lib - 4 # 动态库中变量
0000000000e3  001900000002 R_X86_64_PC32     0000000000000004 global_uninit_val - 4 # 全局变量
0000000000ed  000c00000004 R_X86_64_PLT32    0000000000000000 _Z10global_funv - 4 # 全局方法
000000000102  001a00000004 R_X86_64_PLT32    0000000000000000 _ZN4BeanD1Ev - 4 # Bean 析构方法
000000000120  001a00000004 R_X86_64_PLT32    0000000000000000 _ZN4BeanD1Ev - 4 # Bean 析构方法
000000000129  001b00000004 R_X86_64_PLT32    0000000000000000 _Unwind_Resume - 4  # 异常处理 栈展开
```

*偏移量(Offset)：* `.text`段需要修正的位置(编译时不知道符号最终地址，暂时先用0占位，待连接后知道确切地址时替换成真正地址)。`.rela.text`所以偏移是相对`.text`段的偏移
*信息(Info)：* 低32位(0x00000002)：类型。高32位(0003)：表示在符号表中的索引编号(.symtab)。比如：3对应[符号表](#符号表symtab)中 Num 3 `.L.str`符号。
*类型(Type)：* 重定位条目类型，表示用什么方式计算符号地址填入引用位置，寻找方式。

1. `R_X86_64_PC32`：直接填当前指令(rip指令指针寄存器)地址与符号之间差值，用于同模块内部访问(同一个so库内部，主程序内部)。比如：`mov 0x2ebe(%rip),%eax`
2. `R_X86_64_PLT32`：填该函数在 PLT 中跳转条目的地址，用于访问动态库中方法。比如：`callq  printf@PLT`
3. `R_X86_64_REX_GOTP`：填当前指令(rip)地址与got槽位差值，通过got槽位访问，用于访问动态库变量。比如：`mov  0x2e86(%rip),%rax`,`mov (%rax),%ecx`。
4. `R_X86_64_TPOFF32`：填基于线程控制块地址($fs_base)的偏移，用于访问来自动态库中的线程本地变量(tls)快速模式($fs_base+偏移)。

> 编译器除了生成我们写的正常代码外，还会生成了一些额外的代码。这写代码和异常处理，代码优化(-O3)有关。具体都有些什么不知道啊

### 链接 linker

`clang++ main_test_compile.cpp utils.cpp Bean.cpp -Ltest_lib/build -ltest_lib_shared -o main_test_compile`，预处理 + 编译 + 汇编 + 链接 = 可执行文件
将多个目标文件(.o)和库文件合并成一个可执行文件。（目标文件是.o还是.cpp都可以）

1. 符号合并，将所有符号表(目标文件和库文件)合并成一个大的符号表。比如：main.o需要`_ZN5Utils5getIdEf`但是不知道在哪里(`UND`)，Utils.o说我定义了`_ZN5Utils5getIdEf`。需要`_Z12test_lib_addii`但是不知道在哪里(`UND`)，所有目标文件都扫描完了还是没找到，按顺序扫描依赖的库文件(动态库或者静态库)查找是否定义这个符号。如果目标文件和库文件中都没有找到，报错`undefined reference to ...`。如果在同一个链接单元内找到多个同名的`GLOBAL`符号, 报错`multiple definition of ...`。如此以来，在一个链接单元内，每个符号都对应有唯一一个出处。比如：在 Global.h里写了`int g_val = 10`; A.cpp 包含了它，B.cpp 也包含了它，链接时报错`multiple definition of g_val`。正确做法是Global.h里写`extern int g_val`声明，然后只在仅在一个cpp文件中定义`int g_val = 10`。全工程共享同一个变量，h文件中`extern int val;`声明，cpp文件中`int val = 10`定义。

2. 段合并，将同名的段合并(除了动态库中的段)，并计数出符号的最终地址。比如：目标文件的`.text`段和main.o的`.text`合并。

3. 根据重定位表将引用符号的位置替换成符号真正的地址。对于`R_X86_64_REX_GOTP`类型符号(动态库中的变量)，将会在`.got`段预留一个指针空间，指向运行时符号(变量)，引用符号的位置替换成`.got`段中存储指针的地址。对于`R_X86_64_PLT32`类型符号(动态库中方法)，会在`.got.plt`段预留一个指针空间，指向运行时符号(方法)，还会在代码段生成`@plt`的方法。

> - 编译单元，单个源文件(.cpp) + 包含的头文件(.h) = 目标文件(.o)。编译的最小单位
> - 链接单元，参与链接的文件集合(不包含动态库中文件)。多个目标文件(.o) + 静态库(.a) = 独立二进制文件(可执行文件，动态库，或者静态库)。链接(静态链接ld)的最小单元

目标文件，静态库，动态库都没有找到符号报错`undefined reference to ...`，目标文件，静态库定义同名符号报错`multiple definition of ...`。也就是说动态库中符号，参与了连接时是否定义符号检查，但是没有参与重定义符号检查。这么做的目的是，动态库中符号允许被替换。

#### 符号替换

在编译或者运行时，替换已有实现的方法或变量符号。

1. 链接时，同一连接单元内，强符号(`GLOBAL`)替换弱符号(`WEAK`)

    ```cpp
    /*主程序cpp文件定义 弱符号*/
    namespace Utils {
        __attribute__((weak)) int weak_val = 0x91;

        __attribute__((weak)) void weak_fun() {
            printf("main weak %s %0x\n", __func__, weak_val);
        }
    }
    // 32: 0000000000001140    33 FUNC    WEAK   DEFAULT   15 _ZN5Utils8weak_funEv
    // 36: 0000000000004018     4 OBJECT  WEAK   DEFAULT   25 _ZN5Utils8weak_valE

    /*静态库cpp文件定义 强符号*/
    namespace Utils {
        int weak_val = 0x12;

        void weak_fun() {
            printf("lib GLOBAL %s, weak_val:0x%0x\n", __func__, weak_val);
        }
    }
    // 11: 0000000000000020    33 FUNC    GLOBAL DEFAULT    2 _ZN5Utils8weak_funEv
    // 12: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    5 _ZN5Utils8weak_valE

    int main() {
        // empty(); // 静态库的空方法(有意义的空方法 小心被优化)， 持有静态库强符号。触发静态库连接时展开
        Utils::weak_fun();
    }
    ```

    ```shell
        ➜ clang++ main.cpp libtestlib.a -o main   
        ➜ ./main
        main weak weak_fun 91  # 没有持有对静态库的强引用，连接时静态库未展开。所以静态库中的强符号没有替换 主程序的弱符号


        ➜ clang++ main.cpp -Wl,--whole-archive libtestlib.a -Wl,--no-whole-archive -o main  # 编译时增加--whole-archive参数，静态库展开
        ➜ ./main
        lib GLOBAL weak_fun, weak_val:0x12
    ```

    > 静态库的懒链接，主程序持有对静态库中符号的强引用，或者编译时添加`--whole-archive`，链接时静态库才会展开。来自静态库中的符号才会参与链接。

2. 链接时，来自动态库的符号，被主程序或者静态库(需要展开)中的同名符号替换(动态库中符号，参与是否存在符号的判断。但是没有参与是否重复定义符号的判断)

    ```cpp
    // libm库 double sin(double x);
    double sin(double x) {
        printf("my sin %f\n", x);
        return 0;
    }

    int main() {
        printf("ret %f\n", sin(M_PI_2));
    }
    ```

    ```shell
    ➜ objdump main -d
    00000000000011f0 <main>:
    11fc:       e8 af ff ff ff          call   11b0 <sin> # 主程序覆盖来自libm库的sin方法，RIP(PC)寄存器 + 偏移寻址， 直接寻址调用

    ➜ objdump main -d 
    00000000000011a0 <main>: 
    11ac:       e8 8f fe ff ff          call   1040 <sin@plt> # 没有覆盖来自libm库的sin方法，plt + got 寻址
    ```

    > 动态库中的符号会被主程序中同名符号所覆盖。为了避免意外覆盖，使用命名空间，减小重名几率。使用`static`将符号作用域限制在当前文件内。使用`__attribute__((visibility("hidden")))`将符号作用域限制在二进制文件(so库)内部。

3. 运行时，动态连接器(ld.so)，按查找顺序填充动态库中符号地址。谁先被看到，谁就会被填到`got`，即使先看到的是`WEAK`类型地址，也是使用先看到的符号地址。
运行时查找动态库顺序，`LD_PRELOAD`指定的库 > 主程序 > NEEDED 列表中的依赖库(`readelf main -d`)

    ```cpp
    int main() {
        printf("ret %f\n", sin(M_PI_2));
    }

    // 动态库中定义sin方法
    extern "C" double sin(double x) { // libm C写到
        printf("my sin %f\n", x);
        return 0;
    }
    ```

    ```shell
    ➜ readelf libtestlib.so -s
    22: 00000000000011c0    41 FUNC    GLOBAL DEFAULT   12 sin  # 动态库定义sin符号

    ➜ clang++ main.cpp -o main 

    ➜ readelf main -d                        
    Dynamic section at offset 0x2db0 contains 29 entries:
    标记        类型                         名称/值
    0x0000000000000001 (NEEDED)             共享库：[libstdc++.so.6]
    0x0000000000000001 (NEEDED)             共享库：[libm.so.6]
    0x0000000000000001 (NEEDED)             共享库：[libgcc_s.so.1]
    0x0000000000000001 (NEEDED)             共享库：[libc.so.6]

    ➜ LD_PRELOAD=./libtestlib.so ./main    # LD_PRELOAD指定加载的动态库，运行
    my sin 1.570796 # 使用指定动态库中的sin方法
    ret 0.000000

     ➜ ./main 
    ret 1.000000 # 使用系统库(libm)中的sin方法
    ```

    > 替换动态库符号时，小心递归循环。比如：替换`malloc`方法，并且在替换后的`malloc`方法中调用了`printf`，`printf`内部又会调用`malloc`(当前进程替换后的)，如此以来就形成了递归死循环。内存泄漏检

## 库

预先写好，编译好的代码包。

### 静态库

多个目标文件(.o)，归档

```shell
ar rcs libtestlib.a testlib.o file1.o # ar GNU归档命令。 用于创建，提取静态库文件
```

链接时，以.o文件为单位(一个过大的cpp文件可能被拆分成多个.o)合并到二进制文件(动态库或者可执行文件)。比如：可执行文件中只引用了Utils.o中的一个方法，但是它会将Utils.o文件中的各个段全部合并到二进制文件。通过编译参数将方法和变量单独成一段(`-ffunction-sections -fdata-sections`)，连接时时丢弃未引用的符号(`-Wl,--gc-sections`)。这个过程编译器自动优化也可能触发。

```shell
➜ objdump -d main 
main：     文件格式 elf64-x86-64
0000000000001130 <main>:
    ...
    113f:       bf 02 00 00 00          mov    $0x2,%edi
    1144:       be 03 00 00 00          mov    $0x3,%esi
    1149:       e8 12 00 00 00          call   1160 <_Z12test_lib_addii>
    114e:       31 c0                   xor    %eax,%eax
    1150:       48 83 c4 10             add    $0x10,%rsp
    ...

# 符号表
readelf -s main  

Symbol table '.symtab' contains 40 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
    19: 0000000000001160    18 FUNC    GLOBAL DEFAULT   14 _Z12test_lib_addii # 静态库中的方法
    31: 0000000000004010     4 OBJECT  GLOBAL DEFAULT   24 gloab_lib # 静态库中的变量
    35: 00000000000011b0    44 FUNC    GLOBAL DEFAULT   14 _Z12test_lib_minii # 静态库中的方法
    39: 0000000000001180    44 FUNC    GLOBAL DEFAULT   14 _Z12test_lib_maxii # 静态库中的方法
```

只引用了静态库中的`_Z12test_lib_addii`符号，但是可执行文件中包含了所有来自.cpp文件中定义的符合(方法和变量)。

- 静态库中对应段合并到二进制文件(动态库或者可执行文件)，所以二进制文件体积会变大
- 静态库中符号被多个进程引用，运行时每个进程都会加载一份代码到内存
- 静态库中的符号编译后地址就确定了，直接寻址(rip寄存器 + 偏移)，没有符号查找开销
- 静态库修改用对应的二进制文件就需要重新链接

> [静态库懒链接](#链接-linker)，持有对静态库中符号的强引用，静态库才会在链接时展开，来自静态库符号参与链接。

### 动态库

多个位置无关(pic)目标文件(.o)，链接(ld)成的二进制代码模块。静态库链接时复制代码到可执行文件，动态库是单独加载的二进制模块，并且每次加载位置不同，所以动态库要位置无关的目标文件。动态库中的符号在编译时都不能确定，所以引用动态库中的符号，都将出现在重定位表中。

```shell
# 与clang++ -fpic -shared testlib.cpp -o libtestlib.so 生成的so库不同，clang++ 比 ld 做了更多的事情(依赖库 NEEDED)
ld -shared testlib.o -o libtest_ldlib.so # ld 连接器
```

运行时，每个动态库独立映射(mmap)到内存中。如果一个动态库被多个进程引用，代码(`.text`)在各个进程间共享。变量(`.data`)加载时，在各个进程中虽然虚拟地址不同(ASLR)，但是指向相同物理地址，写变量时copy到进程物理空间。即初始共享，写时复制(cow-copy only write)。

运行时动态库独立映射在内存中，并且每次映射在不同的内存地址。所以在链接时，不知道来自动态库中方法或者变量的地址。也就没办法像静态库一样，直接引用地址。而是通过在`.got`段上预留空间，间接访问来自动态库中的变量。

**.got：** 全局偏移表(Global Offset Table)，存储来自动态库中符号地址，如果引用了来自动态库中的全局变量，全局变量的地址存储在这里。
**.got.plt：** 方法指针数组。数组中每个元素的是指针，指向来自动态库的方法。
**.plt：** 过程链接表(Procedure Linkage Table)，为每个调用动态库中方法，生成的一小段代码(只读)。每次调用动态库中的方法时，都先执行这一小段代码plt[n]。这小段代码写的是，跳转到`.got.plt[n]`提供的地址处执行，第一次提供的地址是连接器，由链接器解析出地址并且回填到`.got.plt[n]`这里，下次调用就是方法的真正地址了。

```shell
# 运行时各个段地址
(gdb) info file
Symbols from "/home/tys/CLionProjects/test/main_test_compile".
Native process:
        Using the running image of child process 170332.
        While running this, GDB does not access memory from...
Local exec file:
        `/home/tys/CLionProjects/test/main_test_compile', file type elf64-x86-64.
        Entry point: 0x555555555090
        ...
        0x0000555555557fb8 - 0x0000555555557fe8 is .got  #运行时 主程序各个段
        0x0000555555557fe8 - 0x0000555555558028 is .got.plt
        0x0000555555558028 - 0x0000555555558048 is .data
        0x0000555555558048 - 0x0000555555558058 is .bss
        ...
        0x00007ffff7ffcfd8 - 0x00007ffff7ffd000 is .got in /lib64/ld-linux-x86-64.so.2 #运行时ld-linux-x86-64.so库加载位置。动态库独立映射在内存中
        0x00007ffff7ffd000 - 0x00007ffff7ffe104 is .data in /lib64/ld-linux-x86-64.so.2
        0x00007ffff7ffe110 - 0x00007ffff7ffe2d8 is .bss in /lib64/ld-linux-x86-64.so.2
        ...
        0x00007ffff7fbbfc8 - 0x00007ffff7fbbfe8 is .got in test_lib/build/libtest_lib_shared.so #运行时libtest_lib_shared.so库加载位置。
        0x00007ffff7fbbfe8 - 0x00007ffff7fbc000 is .got.plt in test_lib/build/libtest_lib_shared.so
        0x00007ffff7fbc000 - 0x00007ffff7fbc00c is .data in test_lib/build/libtest_lib_shared.so
        0x00007ffff7fbc00c - 0x00007ffff7fbc010 is .bss in test_lib/build/libtest_lib_shared.so
        ...
```

```shell
# 动态库代码段 进程间共享
➜ cat /proc/16386/maps  # 只读 可执行 私有 ==> 代码段
71fb127a9000-71fb127aa000 r-xp 00001000 103:0c 9610123                   /home/tys/CLionProjects/thread_local/libutils.so

➜ cat /proc/16318/maps 
7e0eaa630000-7e0eaa631000 r-xp 00001000 103:0c 9610123                   /home/tys/CLionProjects/thread_local/libutils.so

➜ sudo pagemap 16386 0x71fb127a9000 0x71fb127aa000           
0x71fb127a9000     : pfn 4ec7d7           soft-dirty 1 file/shared 1 swapped 0 present 1

➜ sudo pagemap 16318 0x7e0eaa630000 0x7e0eaa631000  # 虚拟地址不同 但是指向相同物理地址     
0x7e0eaa630000     : pfn 4ec7d7           soft-dirty 1 file/shared 1 swapped 0 present 1
```

两个进程引用同一个动态库，代码段虚拟地址不同，但是指向相同物理地址(`4ec7d7`)。

#### 引用动态库中符号

动态库中的符号，要在运行加载后最终地址才确定，因此不能像主程序或者静态库一样，直接寻址(rip寄存器 + 偏移)。而是通过在`.got`段预留指针空间，用来保存运行时来自动态库符号的地址。在程序启动(变量)或者访问(方法)时，填充预留指针空间。用固定位置空间(`.got`)，保存位置不固定符号的地，址访问时到指定位置(位置固定处`.got`)取到地址后，通过地址在访问对应符号。

> 对于来自动态库中的TLS(线程本地)变量，TLS变量每个线程独有，地址不确定。`.got`中保存的是TLS描述符。

##### 引用动态库中的变量

在`.got`段预留指针空间，保存变量地址，程序启动时将变量地址填到预留空间。动态库变量，多进程初始共享，写时复制。

###### 立即绑定 got间接寻址

```cpp
int main() {
    extern int gloab_lib; // 动态库中的变量
    gloab_lib += 1;
    printf("%d \n", gloab_lib);
}
```

```shell
# 引用动态库中的变量 汇编代码
➜ objdump main -d
0000000000001140 <main>:
    1144:       48 8b 05 7d 2e 00 00    mov    0x2e7d(%rip),%rax  # 3fc8 <gloab_lib@Base> # 3fc8地址(got段)处内容mov到rax寄存器
    114b:       8b 08                   mov    (%rax),%ecx # rax寄存器存的地址，该地址处存储的值mov到exc。类似 ecx = *rax
    114d:       83 c1 01                add    $0x1,%ecx                      

➜ objdump -s -j .got main                        
Contents of section .got:
 3fb8 00000000 00000000 00000000 00000000  ................
 3fc8 00000000 00000000 00000000 00000000  ................ # 3fc8 初始值是0
 3fd8 00000000 00000000 00000000 00000000 

# 引用动态库中的变量 gdb运行时
Breakpoint 1, 0x0000555555555144 in main ()  # main断点
(gdb) info files
..
0x0000555555557fb8 - 0x0000555555557fe8 is .got # 主程序got段地址
...
0x00007ffff7fbc008 - 0x00007ffff7fbc014 is .data in test_lib/build/libtest_lib_shared.so 

(gdb) x/a 0x0000555555557fc8
0x555555557fc8: 0x7ffff7fbc010 <gloab_lib> # 1. got段 存储来自动态库变量符号的地址
(gdb) x/w 0x7ffff7fbc010 # 2.通过该地址访问
0x7ffff7fbc010 <gloab_lib>:     0x9
```

###### 初始共享 写时复制

```cpp
// 动态库中，变量定义前后，定义大的数据段，避免写数据时影响到变量所在物理页
char big_buf1[1024 * 4] = {1};
int gloab_lib = 9;
char big_buf2[1024 * 4] = {1};

// main.cpp
int main() {
    extern int gloab_lib;
    printf("pid: %d val:%d %p\n", getpid(), gloab_lib, &gloab_lib); // 动态库变量 修改前
getchar();
    gloab_lib += 1;
    printf("pid: %d val:%d %p\n", getpid(), gloab_lib, &gloab_lib); // 动态库变量 修改后
pause();
}
```

```shell
➜ ./main
pid: 427059 val:9 0x79b7f52e8040
1
pid: 427059 val:10 0x79b7f52e8040

➜ ./main
pid: 427065 val:9 0x786a16613040

# 动态库变量修改前，虚拟地址不同 但是指向相同物理页 pfn 689e86
➜ sudo pagemap 427059 0x79b7f52e8040 0x79b7f52e8044
0x79b7f52e8040     : pfn 689e86           soft-dirty 1 file/shared 1 swapped 0 present 1

➜ sudo pagemap 427065 0x786a16613040 0x786a16613044
0x786a16613040     : pfn 689e86           soft-dirty 1 file/shared 1 swapped 0 present 1

# 动态库变量修改后，指向了新的物理页 pfn 729326
➜ sudo pagemap 427059 0x79b7f52e8040 0x79b7f52e8044
0x79b7f52e8040     : pfn 729326           soft-dirty 1 file/shared 0 swapped 0 present 1
```

> 写时复制，并不是说写这个变量时才复制，而是指写变量所在物理页时复制。初始时进程虚拟地址指向相同物理页，该页在内核中标记只读。写数据时发现只读触发缺页中断，申请新的物理页，复制旧页整页数据到新页，将虚拟地址指向新的页标记为可写。fork进程，父子进程间内存也是初始共享，写时复制。

##### 引用动态库中的方法

和动态库中变量一样，也是在`.got.plt`段预留指针空间，用来保存方法地址，不同的是动态库方法采用懒加载(使用时重定位)。首次调用方法时，预留空间处地址会引导去执行解析函数`_dl_runtime_resolve_xsavec`（来自ld.so），由解析函数查找动态库方法真实地址，返回并回填到预留空间处，下次调用时`.got.plt`段预留空间处就是动态库方法的真实地址。

###### 延迟绑定 plt got寻址

```cpp
int main() {
    int ret = test_lib_add(1,2);
    printf("%d \n", ret);
}
```

```shell
# 引用动态库中的方法 汇编代码
0000000000001020 <printf@plt-0x10>: # plt[0] 动态符号解析函数 _dl_runtime_resolve_xsavec
    1020:       ff 35 ca 2f 00 00       push   0x2fca(%rip)        # 3ff0 <_GLOBAL_OFFSET_TABLE_+0x8>
    1026:       ff 25 cc 2f 00 00       jmp    *0x2fcc(%rip)        # 3ff8 <_GLOBAL_OFFSET_TABLE_+0x10> # .got.plt+0x10
    102c:       0f 1f 40 00             nopl   0x0(%rax)

0000000000001030 <printf@plt>: # printf 桩代码
    1030:       ff 25 ca 2f 00 00       jmp    *0x2fca(%rip)  # 4000 <printf@GLIBC_2.2.5> #.got.plt 首次保存1036，到plt[0]。后续实地址
    1036:       68 00 00 00 00          push   $0x0
    103b:       e9 e0 ff ff ff          jmp    1020 <_init+0x20>

0000000000001040 <_Z12test_lib_addii@plt>: # test_lib_add 桩代码
    1040:       ff 25 c2 2f 00 00       jmp    *0x2fc2(%rip) # 4008 <_Z12test_lib_addii@Base> #.got.plt 首次保存1036，到plt[0]。后续真实地址
    1046:       68 01 00 00 00          push   $0x1
    104b:       e9 d0 ff ff ff          jmp    1020 <_init+0x20>


0000000000001150 <main>:
    ...
    1162:       e8 d9 fe ff ff          call   1040 <_Z12test_lib_addii@plt> # test_lib_add@plt
    ...
    1176:       e8 b5 fe ff ff          call   1030 <printf@plt> # printf@plt

➜ objdump -s -j .got.plt main

Contents of section .got.plt:
 3fe8 a03d0000 00000000 00000000 00000000  .=..............
 3ff8 00000000 00000000 36100000 00000000  ........6....... # 0x0000001036 首次 .got.plt保存下一行指令地址，设置参数到 plt[0]
 4008 46100000 00000000                    F.......         # 0x0000001046 

➜ readelf main -r         
重定位节 '.rela.plt' at offset 0x668 contains 2 entries:
  偏移量          信息           类型           符号值        符号名称 + 加数
000000004000  000100000007 R_X86_64_JUMP_SLO 0000000000000000 printf@GLIBC_2.2.5 + 0    # push   $0x0
000000004008  000200000007 R_X86_64_JUMP_SLO 0000000000000000 _Z12test_lib_addii + 0    # push   $0x1
```

编译时为每个来自动态库中方法，生成了xxx@plt代码(桩代码)，调用动态库中方法就是调用对应的桩代码。桩代码，从指定位置(`got.plt`)取值，跳转到值处去执行。首次这个值是桩代码的下一条指令地址，设置参数(`push $0x0`)执行动态符号解析函数(`_dl_runtime_resolve_xsavec`)。动态符号解析函数找到方法真实地址，返回填写到`got.plt`中，后续调用方法时直接就是方法真实地址。

```shell
# 引用动态库中的方法 调用前后
(gdb) x/a 0x0000555555557fe8+0x18
0x555555558000 <printf@got.plt>:        0x555555555036 <printf@plt+6> # 调用前 桩代码(printf@plt)下一行

(gdb) c
Breakpoint 2, main () at main.cpp:7
(gdb) x/a 0x0000555555557fe8+0x18
0x555555558000 <printf@got.plt>:        0x7ffff7860100 <__printf> # 调用后printf方法真实地址
```

## 线程栈

每个线程创建时分配一段连续的内存空间，默认8M可通过`ulimit -s`查看，这段空间叫线程内存区。这段区域在逻辑上分为，线程控制块(TCB)，线程本地存储区(TLS), 运行时栈(函数栈帧)。

> 栈，一般指所有线程栈集合。

```shell
(gdb) info proc mappings 
process 37126
Mapped address spaces:
          Start Addr           End Addr       Size     Offset  Perms  objfile
          ...
      0x7ffff67fe000     0x7ffff67ff000     0x1000        0x0  ---p   # 保护页 4k
      0x7ffff67ff000     0x7ffff6fff000   0x800000        0x0  rw-p   # 子线程1 8M
      0x7ffff6fff000     0x7ffff7000000     0x1000        0x0  ---p   # 保护页 4k
      0x7ffff7000000     0x7ffff7800000   0x800000        0x0  rw-p   # 子线程2 8M
      0x7ffffffdd000     0x7ffffffff000    0x22000        0x0  rw-p   [stack] # main 线程
```

主线程由内核创建，现在看到的内存大小是0x22000(136k)，它会随着调用而自动增加，最多不会超过8M(`ulimit -s`)。子线程由pthread库创建，创建时预留了8M空间。

```cpp
void funs(int i) {
    //  每次栈上创建1M，消耗栈空间
    printf("cound:%d\n", i);
    char buf[1024 * 1024] = {1};
    funs(i + 1);
}

int main() {
    funs(1);
}
```

```shell
Breakpoint 2, funs (i=4) at main.cpp:20
20    printf("cound:%d\n", i);
(gdb) info proc mappings 
process 40257
Mapped address spaces:
      0x7fffffcfc000     0x7ffffffff000   0x303000        0x0  rw-p   [stack] # i=4 3M

Breakpoint 2, funs (i=5) at main.cpp:20
20    printf("cound:%d\n", i);
(gdb) info proc mappings 
process 40257
Mapped address spaces:
      0x7fffffbfc000     0x7ffffffff000   0x403000        0x0  rw-p   [stack] # i=5 4M
```

### 运行时栈

**rbp寄存器(Base Pointer Register)：** 栈帧基地址寄存器。当前函数栈帧，起始位置(锚点)。

1. 通过偏移访问栈上数据。比如：`movl   $0x0,-0x4(%rbp)`，rbp向下(减)保存局部变量信息，rbp向上(加)保存调用信息(返回地址，第7个参数)。
2. 在函数被调用时，rbp的值会被保存到栈上，然后rbp会指向新的栈帧顶部(x86架构，栈向下增长 从高地址往低地址写)。`push   %rbp`,`mov    %rsp,%rbp`。

**rsp寄存器(Stack Pointer Register)：** 栈指针寄存器。总是指向栈顶，即栈上最后操作的位置。

1. 在执行`push`，`pop`指令时rsp的值自动减小，增加。
2. 栈上空间分配，`sub    $0x10,%rsp` 16字节对齐。

**rip寄存器(Instruction Pointer Register):** 指令指针寄存器(代码段)，ARM叫PC(Program Counter)寄存器。总是指向下一条要执行的指令的地址，cpu根据rip的值取指令出来执行。执行完一条指令后，rip会自动增加，指向下一条指令。

1. `jmp`指令，直接修改rip值，到新的目标地址
2. `call`指令，将下一条指令地址(返回地址)压栈，再修改rip值，到调用函数起始地址。
3. `ret`指令，先从栈顶弹出返回地址，再修改rip值，回到调用的地方，继续向下执行。

```cpp
// main 中调用fun方法
int fun(int x) {
    int local = 12;
    return x + local;
}

int main() {
    fun(1);
    return 0;
}
```

```shell
# main 汇编
Dump of assembler code for function main():
   0x0000555555555150 <+0>: push   %rbp      # 此时rbp是上一个栈帧的基地址(rbp还没赋值)，上一个栈帧基地址压栈
   0x0000555555555151 <+1>: mov    %rsp,%rbp # 当前栈帧基地址
   0x0000555555555154 <+4>: sub    $0x10,%rsp # 栈上分配0x10(16)字节
   0x0000555555555158 <+8>: movl   $0x0,-0x4(%rbp) # 栈上赋值， 代码上看并没有一个为0的局部变量？ 
=> 0x000055555555515f <+15>:    mov    $0x1,%edi # edi 第一个参数
   0x0000555555555164 <+20>:    call   0x555555555130 <_Z3funi> #调用_Z3funi，先下一条指令地址(0x555555555169)入栈，然后在调转到目标位置(0x555555555130)执行
   0x0000555555555169 <+25>:    xor    %eax,%eax # 自己 xor 自己 = 设置0
   0x000055555555516b <+27>:    add    $0x10,%rsp # 栈上空间回收
   0x000055555555516f <+31>:    pop    %rbp # 弹栈，上面push的。取出来并且重新赋值给rbp，所以此时rbp是上一个栈帧的基地址 
   0x0000555555555170 <+32>:    ret # 和call对应的，取call压栈放入的返回地址，跳转到返回地址处向下执行

# fun汇编
Dump of assembler code for function _Z3funi:
   0x0000555555555130 <+0>: push   %rbp     # 压栈 main栈帧基地址
   0x0000555555555131 <+1>: mov    %rsp,%rbp # rbp 当前方法(_Z3funi)栈帧基地址
   0x0000555555555134 <+4>: mov    %edi,-0x4(%rbp)  # 方法参数
   0x0000555555555137 <+7>: movl   $0xc,-0x8(%rbp)  # int local = 12;
   0x000055555555513e <+14>:    mov    -0x4(%rbp),%eax
   0x0000555555555141 <+17>:    add    -0x8(%rbp),%eax # eax 返回值
   0x0000555555555144 <+20>:    pop    %rbp # rbp main栈帧基地址
   0x0000555555555145 <+21>:    ret # 回到 0x0000555555555169继续向下执行
```

#### 栈帧

与函数调用一一对应，当一个函数被调用时，会在栈上分配一段空间(编译结束就知道大小了)，用来存储方法参数，返回地址，局部变量。这块空间就是该函数的栈帧。当函数执行完毕，销毁这个栈帧。rip寄存器回到调用这个函数的地方继续向下执行。当函数调用过深，不停消耗栈空间，超过了栈大小(8M)就会出现栈溢出。

##### 栈帧布局

$rbp+0x10 在这个地址处存储着, 第7个参数参数,如果有的话。参数个数小于7时，寄存器传参，大于等于7个参数部分通过栈传参
$rbp+0x08 在这个地址处存储着，当前栈帧返回地址(调用位置下一条指令地址)
$rbp 当前栈帧基地址，值是0x7fffffffd120。 在这个地址处存储着，调用者基地址(0x7fffffffd140)即，上一个栈帧基地址
$rbp-0x04 计数中间变量
$rbp-0x08 当前方法 局部变量

> $rbp+位置，存储调用者信息(返回地址，调用者rbp)，$rbp-位置，存储当前方法栈帧变量。栈上存储变量有填充对齐。

地址        值
$rbp+0x28: 0x7ffff782a1ca  ← 调用者 返回地址(main 返回地址)， <__libc_start_call_main+122> 调用了main方法
$rbp+0x20: 0x7fffffffd1e0  ← 调用者 rbp地址处,调用者(main栈帧基地址)
$rbp+0x18: 0xffffd268
$rbp+0x10: 0x7ffff7e7aa50
$rbp+0x08: 0x555555555169  ← 返回地址（main+25）

$rbp+0x00: 0x7fffffffd140  ← 调用者的 rbp（上一个栈帧基址 main栈帧基地址） 当前栈帧值

$rbp-0x04: 0x1             ← _Z3funi 参数
$rbp-0x08: 0xc             ← _Z3funi 局部变量 int local = 12;

```shell
# fun方法栈帧附近
Breakpoint 2, fun (x=1) at main.cpp:9
9    int local = 12;
(gdb) p $rbp
$2 = (void *) 0x7fffffffd120  # _Z3funi方法 栈帧基地址值

(gdb) x/a $rbp  # 当前栈帧基地址处保存 上一个栈帧(main)基地址
0x7fffffffd120: 0x7fffffffd140

(gdb) x/a $rbp+8        # 栈帧基地址 + 8 地址处保存着_Z3funi方法 返回地址(0x555555555169) 
0x7fffffffd128: 0x555555555169 <main()+25>


Breakpoint 2, fun (x=1) at main.cpp:9  # fun方法栈帧
9       int local = 12;
(gdb) p $rbp
$2 = (void *) 0x7fffffffd120 # 栈帧基地址值

(gdb) x/8a $rbp # $rbp+ 开始的8个地址
0x7fffffffd120: 0x7fffffffd140  0x555555555169 <main()+25> 
0x7fffffffd130: 0x7ffff7e7aa50 <_ZNSt8ios_base4Init11_S_refcountE>  0xffffd268
0x7fffffffd140: 0x7fffffffd1e0  0x7ffff782a1ca <__libc_start_call_main+122>
0x7fffffffd150: 0x8 0x7fffffffd268

# $rbp-
(gdb) x/w $rbp-4
0x7fffffffd11c: 0x1 # 参数
(gdb) x/w $rbp-8
0x7fffffffd118: 0xc # 局部变量
```

### 线程控制块(thread contol block)

每个线程有自己的tcb区域，创建时分配，是访问线程本地变量(tls)的入口。通过fs寄存器可快速定位到这区域。

```shell
(gdb) p/x $fs_base
$5 = 0x7ffff77ff6c0

(gdb) x/2a $fs_base
0x7ffff77ff6c0: 0x7ffff77ff6c0 0x55555556b2e0
```

$fs_base+0x8 ← DTV数组指针
$fs_base ← 指向自己指针

DTV(Dynamic Thread Vector)，数组，数组每个元素是指针(占用0x10(16)字节)，指向某个模块线程本地存储区域(tls)基地址。

```shell
(gdb) x/a $fs_base+8
0x7ffff77ff6c8: 0x55555556b2e0 # DTV数组指针
(gdb) x/8a 0x55555556b2e0
0x55555556b2e0: 0x1 0x0
0x55555556b2f0: 0x7ffff77ff6b4  0x0  # 模块1(主程序) tls变量存储基地址
0x55555556b300: 0x7ffff77ff6ac  0x0  # 模块2(xxx.so) tls变量存储基地址   
0x55555556b310: 0x7ffff77ff688  0x0  # 模块3(yyyy.so) tls变量存储基地址   
```

### 线程本地存储区(thread local storage)

每个线程私有的，独立的数据存储区域。即使这些数据被声明为全局的，静态的，局部的，也是每个线程有自己的独立副本。`errno`返回的就是线程本地变量。

- tls变量编译后，会形成一个独立`.tdata`段或`.tbss`段。
- 每个tls变量编译后，会增加一个`_ZTW变量名`(Thread Wrapper)线程包装函数。如果变量有定义构造函数会增加一个`_ZTW变量名`(Thread Helper/Initialization)线程初始化辅助函数

线程包装函数(`_ZTWXXX`)：获取tls变量地址
线程初始化辅助函数(`_ZTWXXX`)：判断对象是否已经构建(在`$fs-offset`保存标志位)，没有则调用构造函数，将对象创建到$fs-offset地址处。有定义构造函数才会生成，

```shell
# _ZTW thread_local int tlsVal = 0x01; tls变量地址获取，通过fs寄存器(段寄存器)访问
00000000000011f0 <_ZTW6tlsVal>:
    ...
    11f4:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax # 通过fs寄存器访问 tls变量
    11fb:       00 00 
    11fd:       48 8d 80 e8 ff ff ff    lea    -0x18(%rax),%rax # $fs-0x18 
    ...

# _ZTW thread_local TlsBeanWithCon tlsBeanWithCon; tls变量地址获取
00000000000011d0 <_ZTW14tlsBeanWithCon>:
    11d4:       e8 c7 ff ff ff          call   11a0 <_ZTH14tlsBeanWithCon> # 如果有定义构造函数，调用_ZTH 线程初始化辅助函数
    11d9:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax
    11e0:       00 00 
    11e2:       48 8d 80 f4 ff ff ff    lea    -0xc(%rax),%rax
    ...

# _ZTH tls变量是否已经创建，如果没有则调用构造函数，将对象创建到目标位置($fs-offset)
00000000000011a0 <_ZTH14tlsBeanWithCon>:
    11a4:       64 8a 04 25 fc ff ff    mov    %fs:0xfffffffffffffffc,%al # $fs-0x4地址处值 赋值给 al
    11ab:       ff 
    11ac:       3c 00                   cmp    $0x0,%al # $fs-0x4地址处值是否为0
    11ae:       0f 85 0e 00 00 00       jne    11c2 <_ZTH14tlsBeanWithCon+0x22> # jump no equal, 不是0， 跳过构造函数
    11b4:       64 c6 04 25 fc ff ff    movb   $0x1,%fs:0xfffffffffffffffc # 是0， 这个位置写1
    11bb:       ff 01 
    11bd:       e8 7e fe ff ff          call   1040 <__cxx_global_var_init> # 是个标签，调用到构造函数

0000000000001040 <__cxx_global_var_init>:
    1044:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax
    104b:       00 00 
    104d:       48 8d b8 f4 ff ff ff    lea    -0xc(%rax),%rdi # 对象创建到 $fs-0xc位置
    1054:       e8 d7 01 00 00          call   1230 <_ZN14TlsBeanWithConC1Ev>
    ...
```

#### 主程序中tls变量

##### 主程序中tls变量寻址

没有定义构造函数的tls变量，线程创建时复制`.tdata`数据到线程目标位置，`.tbss`数据清零。有定义构造函数，每次访问时判断对象是否已经构建(编译器生成的线程包装函数`_ZTWxxx`)，首次访问时在目标位置($fs + offset)处调用构造函数创建对象(编译器生成的线程初始化辅助函数`_ZTHxxx`)。通过fs寄存器 + 偏移访问。

```cpp
thread_local int tlsVal = 0x01;
thread_local TlsBean tlsBean; // 没有定义构造函数
thread_local TlsBeanWithCon tlsBeanWithCon; // 定义了无参构造函数，对象创建时调用构造函数,所以编译时值是不确定的 bss段

int main() {
    int tls = tlsVal;
    TlsBean *tls_bean = &tlsBean;
    TlsBeanWithCon *tls_bean_c = &tlsBeanWithCon;
    return 0;
}
```

```shell
0000000000001150 <main>:
    ... 
    115f:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax
    1166:       00 00 
    1168:       48 8d 80 e8 ff ff ff    lea    -0x18(%rax),%rax # $fs-0x18 tlsVal
    116f:       8b 00                   mov    (%rax),%eax
    1171:       89 45 f8                mov    %eax,-0x8(%rbp) #  int tls = tlsVal
    1174:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax
    117b:       00 00 
    117d:       48 8d 80 ec ff ff ff    lea    -0x14(%rax),%rax # $fs-0x14 tlsBean
    1184:       48 89 45 f0             mov    %rax,-0x10(%rbp) # TlsBean *tls_bean = &tlsBean
    1188:       e8 43 00 00 00          call   11d0 <_ZTW14tlsBeanWithCon> # 获取tlsBeanWithCon的地址，如果没有创建则创建
    118d:       48 89 45 e8             mov    %rax,-0x18(%rbp) # TlsBeanWithCon *tls_bean_c = &tlsBeanWithCon;
    ...

00000000000011d0 <_ZTW14tlsBeanWithCon>:
    11d4:       e8 c7 ff ff ff          call   11a0 <_ZTH14tlsBeanWithCon> # 目标位置处创建对象 $fs-0xc, 如果还没有创建对象的话
    11d9:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax
    11e0:       00 00 
    11e2:       48 8d 80 f4 ff ff ff    lea    -0xc(%rax),%rax
    ...

00000000000011a0 <_ZTH14tlsBeanWithCon>:
    11a4:       64 8a 04 25 fc ff ff    mov    %fs:0xfffffffffffffffc,%al # $fs-0x4 是否已经创建标志位
    11ab:       ff 
    11ac:       3c 00                   cmp    $0x0,%al
    11ae:       0f 85 0e 00 00 00       jne    11c2 <_ZTH14tlsBeanWithCon+0x22> # 已经构建，跳过构造函数
    11b4:       64 c6 04 25 fc ff ff    movb   $0x1,%fs:0xfffffffffffffffc
    11bb:       ff 01 
    11bd:       e8 7e fe ff ff          call   1040 <__cxx_global_var_init> # 构造函数，目标位置处创建对象
    ...

 0000000000001040 <__cxx_global_var_init>:
    1044:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax
    104b:       00 00 
    104d:       48 8d b8 f4 ff ff ff    lea    -0xc(%rax),%rdi
    1054:       e8 d7 01 00 00          call   1230 <_ZN14TlsBeanWithConC1Ev>
    ...
```

没有定义构造函数的tls变量，线程创建时加载值到目标位置($fs-offset)。定义了构造函数的tls变量，首次访问时创建对象到目标位置($fs-offset)

```shell
Temporary breakpoint 2, main () at main.cpp:10
10          int tls = tlsVal;
(gdb) x/w $fs_base-0x18
0x7ffff7e8f528: 1 # 没有定义构造函数的tls变量，上来值就已经有了
(gdb) x/2x $fs_base-0x14
0x7ffff7e8f52c: 0x00000021      0x00000022
(gdb) x/w $fs_base-0x4 # 定义构造函数的tls变量，未访问前，是否已经创建标志位0 值也是0
0x7ffff7e8f53c: 0x00000000
(gdb) x/2w $fs_base-0xc
0x7ffff7e8f534: 0x00000000      0x00000000

Breakpoint 1, main () at main.cpp:13 
13          return 0;
(gdb) x/w $fs_base-0x4 # 定义构造函数的tls变量，访问后，标志位1 值有了
0x7ffff7e8f53c: 0x00000001
(gdb) x/2w $fs_base-0xc
0x7ffff7e8f534: 0x00000011      0x00000012
```

#### 动态库中tls变量

##### tls描述符

在动态库的`.got`段上，为每个tls变量，生成tls描述信息。描述信息中包括模块id(`R_X86_64_DTPMOD64`)和偏移(`R_X86_64_DTPOFF64`)。模块id，唯一标识一个动态库，是[线程DTV数组](#线程控制块thread-contol-block)下标。偏移，基于线程tls基地址的偏移($fs)。在程序启动时搜集所有tls变量，为动态库中的tls变量分配模块id，规划每个tls变量存储区域(在线程本地存储区域中的位置)。也就是说tls描述符，在程序启动时规划填充好值。

```shell
➜ readelf libTestBinlib.so -r

重定位节 '.rela.dyn' at offset 0x568 contains 14 entries:
  偏移量          信息           类型           符号值        符号名称 + 加数
...
000000003fa0  000a00000010 R_X86_64_DTPMOD64 0000000000000004 tlsBean + 0  # tlsBean tls描述符
000000003fa8  000a00000011 R_X86_64_DTPOFF64 0000000000000004 tlsBean + 0
000000003fb0  000b00000010 R_X86_64_DTPMOD64 000000000000000c tlsBeanWithCon + 0 # tlsBeanWithCon tls描述符
000000003fb8  000b00000011 R_X86_64_DTPOFF64 000000000000000c tlsBeanWithCon + 0
000000003fd8  000800000010 R_X86_64_DTPMOD64 0000000000000000 tlsVal + 0 # tlsVal tls描述符， int 类型
000000003fe0  000800000011 R_X86_64_DTPOFF64 0000000000000000 tlsVal + 0

objdump -s -j .got libTestBinlib.so
Contents of section .got:
 3f88 00000000 00000000 00000000 00000000  ................
 3f98 00000000 00000000 00000000 00000000  ................
 3fa8 00000000 00000000 00000000 00000000  ................
 3fb8 00000000 00000000 00000000 00000000  ................
 3fc8 00000000 00000000 00000000 00000000  ................


(gdb) info files 
0x00007ffff7fbbf88 - 0x00007ffff7fbbfe8 is .got in ./libTestBinlib.so

(gdb) x/2a 0x00007ffff7fbbf88+0x18 # tlsBean tls描述符
0x7ffff7fbbfa0: 0x1 0x4
(gdb) x/2a 0x00007ffff7fbbf88+0x28 # tlsBeanWithCon tls描述符
0x7ffff7fbbfb0: 0x1 0xc
(gdb) x/2a 0x00007ffff7fbbf88+0x50 # tlsVal tls描述符
0x7ffff7fbbfd8: 0x1 0x0
```

##### 动态库中tls变量寻址

对动态库中tls变量访问，通过线程包装函数(`_ZTWxxx`)。线程包装函数中判断是否存在线程初始化辅助函数(`_ZTHxxx`)，变量定义了构造函数，则存在线程初始化辅助函数，否则没有线程辅助函数。对于没有定义构造函数的tls变量，在`.got`上保存每个tls变量的偏移(线程创建时填写)，fs寄存器+偏移直接访问。对于有定义构造函数的tls变量，每次访问都会调用线程辅初始化助函数(`_ZTHxxx`)，在线程辅助初始化助函数中，判断对象是否已经创建(tls存储位置附近有个标志位)，没有创建则通过tls描述符和`__tls_get_addr`函数获取目标存储地址，调用构造函数，在目标位置处创建对象。

> `__tls_get_addr`函数内部，通过线程DTV数组和tls描述符获取目标位置。线程DTV数组[tls描述符.模块id] + ls描述符.偏移 = tls变量地址

```shell
0000000000001140 <main>:
    114f:       e8 2c 00 00 00          call   1180 <_ZTW6tlsVal> # thread_local int tlsVal = 0x01;
    1154:       8b 00                   mov    (%rax),%eax
    1156:       89 45 f8                mov    %eax,-0x8(%rbp)
    1159:       e8 62 00 00 00          call   11c0 <_ZTW7tlsBean> # thread_local TlsBean tlsBean; 没有定义构造函数
    115e:       48 89 45 f0             mov    %rax,-0x10(%rbp)

    1162:       e8 99 00 00 00          call   1200 <_ZTW14tlsBeanWithCon> # thread_local TlsBeanWithCon tlsBeanWithCon{}; 定义了无参构造函数
    1167:       48 89 45 e8             mov    %rax,-0x18(%rbp)
    ...

# thread_local int tlsVal = 0x01;
0000000000001180 <_ZTW6tlsVal>:
    1184:       48 8b 05 25 2e 00 00    mov    0x2e25(%rip),%rax        # 3fb0 <_ZTH6tlsVal@Base> # .got， 有构造函数地址就有值，指向so库中 _ZTH6tlsVal 方法
    118b:       48 85 c0                test   %rax,%rax
    118e:       0f 84 0a 00 00 00       je     119e <_ZTW6tlsVal+0x1e> # rax等于0 没有构造函数
    1194:       e9 00 00 00 00          jmp    1199 <_ZTW6tlsVal+0x19>
    1199:       e8 a2 fe ff ff          call   1040 <_ZTH6tlsVal@plt> 
    119e:       48 8b 0d 3b 2e 00 00    mov    0x2e3b(%rip),%rcx        # 3fe0 <tlsVal@Base> # .got， 基于$fs的偏移
    11a5:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax
    11ac:       00 00 
    11ae:       48 01 c8                add    %rcx,%rax # rax = $fs + offset
    ...

# thread_local TlsBean tlsBean; 没有定义构造函数 
00000000000011c0 <_ZTW7tlsBean>:
    11c4:       48 8b 05 ed 2d 00 00    mov    0x2ded(%rip),%rax        # 3fb8 <_ZTH7tlsBean@Base> # 没定义构造函数类型 和基础类型是一样的
    11cb:       48 85 c0                test   %rax,%rax
    11ce:       0f 84 0a 00 00 00       je     11de <_ZTW7tlsBean+0x1e> # 没有定义构造函数
    11d4:       e9 00 00 00 00          jmp    11d9 <_ZTW7tlsBean+0x19>
    11d9:       e8 6a fe ff ff          call   1048 <_ZTH7tlsBean@plt>
    11de:       48 8b 0d c3 2d 00 00    mov    0x2dc3(%rip),%rcx        # 3fa8 <tlsBean@Base>
    11e5:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax
    11ec:       00 00 
    11ee:       48 01 c8                add    %rcx,%rax
    ...

# thread_local TlsBeanWithCon tlsBeanWithCon{}; // 定义了无参构造函数
0000000000001200 <_ZTW14tlsBeanWithCon>:
    1204:       48 8b 05 95 2d 00 00    mov    0x2d95(%rip),%rax        # 3fa0 <_ZTH14tlsBeanWithCon@Base> # 保存动态库中 _ZTH14tlsBeanWithCon 方法地址 
    120b:       48 85 c0                test   %rax,%rax # 是不是0
    120e:       0f 84 0a 00 00 00       je     121e <_ZTW14tlsBeanWithCon+0x1e>
    1214:       e9 00 00 00 00          jmp    1219 <_ZTW14tlsBeanWithCon+0x19>
    1219:       e8 1a fe ff ff          call   1038 <_ZTH14tlsBeanWithCon@plt> # 有构造函数，执行so库中的 _ZTH14tlsBeanWithCon 方法
    121e:       48 8b 0d 9b 2d 00 00    mov    0x2d9b(%rip),%rcx        # 3fc0 <tlsBeanWithCon@Base>
    1225:       64 48 8b 04 25 00 00    mov    %fs:0x0,%rax
    122c:       00 00 
    122e:       48 01 c8                add    %rcx,%rax
    1231:       5d                      pop    %rbp
    1232:       c3                      ret

# .so
0000000000001170 <_ZTH14tlsBeanWithCon>:
    1174:       48 8d 3d 0d 2e 00 00    lea    0x2e0d(%rip),%rdi        # 3f88 <_DYNAMIC+0x200> # 动态库中tls变量描述符
    117b:       e8 d0 fe ff ff          call   1050 <__tls_get_addr@plt>
    1180:       8a 80 14 00 00 00       mov    0x14(%rax),%al
    1186:       3c 00                   cmp    $0x0,%al # 是否已经创建
    1188:       0f 85 18 00 00 00       jne    11a6 <_ZTH14tlsBeanWithCon+0x36> # 已经创建 跳到 11a6
    118e:       48 8d 3d f3 2d 00 00    lea    0x2df3(%rip),%rdi        # 3f88 <_DYNAMIC+0x200> # 动态库中tls变量描述符
    1195:       e8 b6 fe ff ff          call   1050 <__tls_get_addr@plt>
    119a:       c6 80 14 00 00 00 01    movb   $0x1,0x14(%rax)
    11a1:       e8 ca fe ff ff          call   1070 <__cxx_global_var_init> # 还没创建
    11a6:       5d                      pop    %rbp
    11a7:       c3                      ret
    11a8:       0f 1f 84 00 00 00 00    nopl   0x0(%rax,%rax,1)
    11af:       00 

0000000000001070 <__cxx_global_var_init>:
    1074:       66 48 8d 3d 34 2f 00    data16 lea 0x2f34(%rip),%rdi        # 3fb0 <tlsBeanWithCon@@Base+0x3fa4> # so库中，tlsBeanWithCon tls变量描述符
    107b:       00 
    107c:       66 66 48 e8 cc ff ff    data16 data16 rex.W call 1050 <__tls_get_addr@plt> # 线程DTV数组[模块id] + 偏移 = tls变量地址
    1083:       ff 
    1084:       48 89 c7                mov    %rax,%rdi # rax __tls_get_addr返回地址，创建到rax处。线程目标地址处($fs + offset)创建
    1087:       e8 b4 ff ff ff          call   1040 <_ZN14TlsBeanWithConC1Ev@plt> # 构造函数 目标位置处构建对象
    ... 
```

> `thread_local`定义tls变量和线程特定数据区域(`pthread_key_create`,`pthread_setspecific`)完全不同。`thread_local`是cpp关键字，依赖编译器生成线程包装函数(`_ZTWxxx`)和线程初始化辅助函数(`ZTHxxx`)，启动时规划好每个tls变量在线程本地存储区域中的位置(数据在栈上tls段)，通过线程fs寄存器 + 偏移访问。线程特定数据区域通过一些列函数(创建/销毁key，set/get)，启动时根据key为每个线程，在线程本地存储区域中规划好了一个指针数组(key是数组下标)，`pthread_setspecific`方法，保存一个指针数据到线程本地存储区域，`pthread_getspecific`获取设置的指针值。

## 匿名函数

匿名函数，也叫Lambda表达式，是一种没有名称的函数，通常用于参数传递，小块逻辑封装。

```cpp
int main() {
    int x = 0x20, y = 0x10;
    auto f = [x, y](int) -> int { return 0; };
    int ret = f(0x99);
    return 0;
}
```

*[]*，捕获列表，值传递，引用传递，指针传递。值捕获`[x, y]`，引用捕获`[&x, &y]`，所有变量值捕获`[=]`，所有变量引用捕获`[&]`。
*()*，方法参数列表。`(int)`
*-> int*，方法返回值，void时可以省略。`-> int`
*{}*，方法体。`{ return 0;}`

```shell
0000000000001140 <main>:
    114f:       c7 45 f8 20 00 00 00    movl   $0x20,-0x8(%rbp)
    1156:       c7 45 f4 10 00 00 00    movl   $0x10,-0xc(%rbp)
    115d:       8b 45 f8                mov    -0x8(%rbp),%eax
    1160:       89 45 ec                mov    %eax,-0x14(%rbp)
    1163:       8b 45 f4                mov    -0xc(%rbp),%eax
    1166:       89 45 f0                mov    %eax,-0x10(%rbp) # -0x8(%rbp)至-0xc(%rbp)值 复制到 -0x14(%rbp)至-0x10(%rbp) 值传递
    1169:       48 8d 7d ec             lea    -0x14(%rbp),%rdi # 第一个参数this(类成员变量开始地址处)
    116d:       be 99 00 00 00          mov    $0x99,%esi # 方法参数
    1172:       e8 19 00 00 00          call   1190 <_ZZ4mainENK3$_0clEi>
    1177:       89 45 e8                mov    %eax,-0x18(%rbp) # 方法返回值
    117a:       31 c0                   xor    %eax,%eax
    117c:       48 83 c4 20             add    $0x20,%rsp
    1180:       5d                      pop    %rbp
    1181:       c3                      ret
    1182:       66 66 66 66 66 2e 0f    data16 data16 data16 data16 cs nopw 0x0(%rax,%rax,1)
    1189:       1f 84 00 00 00 00 00 

0000000000001190 <_ZZ4mainENK3$_0clEi>:
    1194:       48 89 7d f8             mov    %rdi,-0x8(%rbp) # this 参数
    1198:       89 75 f4                mov    %esi,-0xc(%rbp) # 方法参数
    119b:       31 c0                   xor    %eax,%eax # 返回值

# 方法名解析
➜ c++filt "_ZZ4mainENK3\$_0clEi" 
main::$_0::operator()(int) const
```

匿名函数相当于在方法内部建立了一个叫`$_0`的类，捕获列表是类构造方法参数(值捕获，构造方法中参数就是值。引用捕获，构造方法中参数就是引用类型)，捕获值传递到类成员变量。调用匿名方法时，实例化`$_0`类，调用类的仿函数，调用类方法第一个参数是this。

```cpp
int main{
    // 指针函数
    // int (*pFun)(int) = fun;

    int x = 0x20, y = 0x10;
    struct $_0 {
        $_0(int captureVal1, int captureVal2) : capture_val1(captureVal1), capture_val2(captureVal2) {}

        int operator()(int arg) const { // 默认是const，不能修改捕获变量的值， 可以用 mutable
            return 0;
        }
        int capture_val1;
        int capture_val2;
    };
    $_0 f(x, y);
    int ret = f(0x99);
    return 0;
}
```

> 函数指针`int (*pFun)(int)`和匿名方法功能类似。函数指针在编译时，只知道这个是代码首地址没办法内联。匿名方法，编译时调用方法内部指令是确定的方便内联，减少函数调用开销。所以匿名函数通常比指针函数性能更快。



