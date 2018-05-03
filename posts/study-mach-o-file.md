# 探秘 Mach-O 文件

之前负责项目的包体积优化学习了 Mach-O 文件的格式，那么 Mach-O 究竟是怎么样的文件，知道它的组成之后我们又能做点什么？本文会从 Mach-O 文件的介绍讲起，再看看认识它后的一些实际应用。

由于本文篇幅有点长，这里添加了文章导航方便阅读

* [Mach-O 文件格式](#mach-o 文件格式)
* [减少包大小][1]
* [获取调用堆栈][2]
* [如何用 MachO 文件关联类的方法名](#如何用 macho 文件关联类的方法名)

### Mach-O 文件格式

先让我们看看 Mach-O 的大致构成

![][image-1]

再使用 MachOView 一窥究竟

![][image-2]

结合可知 Mach-O 文件包含了三部分内容：

- Header（头部），指明了 cpu 架构、大小端序、文件类型、Load Commands 个数等一些基本信息
- Load Commands（加载命令)，正如官方的图所示，描述了怎样加载每个 Segment 的信息。在 Mach-O 文件中可以有多个 Segment，每个 Segment 可能包含一个或多个 Section。
- Data（数据区），Segment 的具体数据，包含了代码和数据等。

#### Headers
Mach-O 文件的头部定义如下：

![][image-3]

- magic 标志符 0xfeedface 是 32 位， 0xfeedfacf 是 64 位。
- cputype 和 cpusubtype 确定 cpu 类型、平台
- filetype 文件类型，可执行文件、符号文件（DSYM）、内核扩展等
- ncmds 加载 Load Commands 的数量
- flags dyld 加载的标志
	- `MH_NOUNDEFS` 目标文件没有未定义的符号，
	- `MH_DYLDLINK` 目标文件是动态链接输入文件，不能被再次静态链接, 
	- `MH_SPLIT_SEGS` 只读 segments 和 可读写 segments 分离，
	- `MH_NO_HEAP_EXECUTION` 堆内存不可执行…

filetype 的定义有：

![][image-4]

flags 的定义有：

![][image-5]

简单总结一下就是 Headers 能帮助校验 Mach-O 合法性和定位文件的运行环境。
#### Load Commands
Headers 之后就是 Load Commands，其占用的内存和加载命令的总数在 Headers 中已经指出。

![][image-6]

Load Commands 的定义比较简单：

![][image-7]

- cmd 字段，如上图它指出了 command 类型
	- `LC_SEGMENT、LC_SEGMENT_64 ` 将 segment 映射到进程的内存空间，
	- `LC_UUID ` 二进制文件 id，与符号表 uuid 对应，可用作符号表匹配，
	- `LC_LOAD_DYLINKER` 启动动态加载器，
	- `LC_SYMTAB ` 描述在 `__LINKEDIT ` 段的哪找字符串表、符号表，
	- `LC_CODE_SIGNATURE` 代码签名等
- cmdsize 字段，主要用以计算出到下一个 command 的偏移量。

#### Segment & Section
这里先来看看 segment 的定义：

![][image-8]

- cmd 就是上面分析的 command 类型
- segname 在源码中定义的宏
	- `#define  SEG_PAGEZERO    "__PAGEZERO" // 可执行文件捕获空指针的段 `
	- `#define  SEG_TEXT    "__TEXT" // 代码段，只读数据段 `
	- `#define  SEG_DATA    "__DATA"    // 数据段 `
	- `#define  SEG_LINKEDIT    "__LINKEDIT" // 包含动态链接器所需的符号、字符串表等数据 `
- vmaddr 段的虚存地址（未偏移），由于 ALSR，程序会在进程加上一段偏移量（slide），真实的地址 = vm address + slide
- vmsize 段的虚存大小
- fileoff 段在文件的偏移
- filesize 段在文件的大小
- nsects 段中有多少个 section

接着看看 section 的定义：

![][image-9]

`__Text` 和 `__Data` 都有自己的 section

- segname 就是所在段的名称
- sectname section名称，部分列举：
	- `Text.__text` 主程序代码
	- `Text.__cstring` c 字符串
	- `Text.__stubs` 桩代码
	- `Text.__stub_helper` 
	- `Data.__data` 初始化可变的数据
	- `Data.__objc_imageinfo` 镜像信息 ，在运行时初始化时 `objc_init`，调用 `load_images` 加载新的镜像到 infolist 中![][image-10]
	- `Data.__la_symbol_ptr` 
	- `Data.__nl_symbol_ptr` 
	- `Data.__objc_classlist` 类列表
	- `Data.__objc_classrefs` 引用的类

这节最后探究下 stubs，在 Xcode 中新建 C 项目，代码如下：
	#include <stdio.h>
	int main(int argc, const char * argv[]) {
	    printf("Hello, coder\n");
	    return 0;
	}

使用 `gcc -c main.c` 将其编译成 a.out 文件，调用 nm 命令查看 .o 文件的符号

![][image-11]

看到 `_printf ` 是未定义的，也就是说并没有该函数的内存地址。nm 打印出的信息表明`dyld_stub_binder ` 也是未定义的。
打开 Hopper 查看 .o 文件

![][image-12]

可以看出 printf 会跳入 `__stubs` 中，地址也与 MachOView 看到的相对应

![][image-13]

双击刚才  `__stubs`  中的地址，会跳转到 `__la_symbol_ptr`

![][image-14]

在 MachOView 中查看 0x100001010 对应的数据为 0x10000f9c

![][image-15]

用 Hopper 搜索 0x10000f9c，跳转到 `stub_helper`，可知 `__la_symbol_ptr` 里的数据被 bind 成了  `stub_helper`

![][image-16]

由此可知，`__la_symbol_ptr` 中的数据被第一次调用时会通过 `dyld_stub_binder` 进行相关绑定，而 `__nl_symbol_ptr` 中的数据就是在动态库绑定时进行加载。

![][image-17]

所以 `__la_symbol_ptr` 中的数据在初始状态都被 bind 成  `stub_helper`，接着 `dyld_stub_binder ` 会加载相应的动态链接库，执行具体的函数实现，此时 `__la_symbol_ptr` 也获取到了函数的真实地址，完成了一次近似懒加载的过程。  

写到这里，算是快速过了一遍 Mach-O 文件的基本概念，接着聊聊可以怎样减少项目的体积。

<h3 id="减少包大小">减少包大小</h3>

iOS 的包主要由可执行文件、资源文件（图片）等文件组成，所以可以从这两大头文件入手优化。

#### 可执行文件瘦身

我们的项目中难免会存在一些没使用的类或方法，由于 OC 的动态特性，编译器会对所有的源文件进行编译，找出并删除没用到的类或方法可以减少可执行文件大小。

上文中提到了 `__objc_classlist` 和 `__objc_classrefs `，它们分别表示项目中全部类列表和项目中被引用的类列表，那么取两者之差，就能删除一些项目中没使用的类文件。但是在删除过程中记住要在项目中全局搜索确认下，看看有没有通过字符串调用无引用的类的方法，原因还是 OC 是动态语言。
在看具体做法之前，顺带提一下我公司的项目组成。我们维护着俩客户端，共用着一个基础库（lib 库），可能有时由于产品的需求变更或者为了产品功能的预留导致 lib 库中只有着某个端使用的代码，我在上述的做法中对脚本做了稍微改进，以防删除了 lib 库的代码，导致另一个端跑不起来，下面介绍通用的做法：

- 在控制台输入 `otool -v -s __objc_classlist ` 和 `otool -v -s __objc_classrefs ` 命令，逆向 `__DATA. __objc_classlist` 段和 `__DATA. __objc_classrefs ` 段获取当前所有oc类和被引用的oc类。
- 取两者差集，得到没被引用的类的段地址
- otool -o 二进制文件，获取段信息
- 通过脚本使用没被引用的类的段地址去段信息中匹配出具体类名

#### 压缩图片资源
这点就跟本文的主题没什么关系，不感兴趣可以略过。
压缩 app 中的图片是我做的另一个努力，虽然 Xcode 会压一遍，但是经我压缩后打包发现包还是会少个将近 1m，这里用到的工具是 ImageOptim，贴出我的三脚猫 python：

```py
all_file_size = 0
all_file_count = 0
	
def fileDriector(filePath):
    global all_file_size, all_file_count
	
    for file in os.listdir(filePath):
        if os.path.isdir(filePath + '/' + file):
            if file != 'Pods' and not file.startswith('.') and not file.endswith('.framework') \
                    and not file.endswith('.bundle') and not file.endswith('.a') and file != 'libs' \
                    or file.endswith('.xcassets') or file.endswith('.imageset'):
                the_path = filePath + '/' + file
                fileDriector(the_path)
        elif file.endswith('.png') or file.endswith('.jpg'):
            fileName = filePath + '/' + file
	
            comand_line = "echo %s | imageoptim" % fileName
            test = subprocess.Popen(comand_line, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
            output = test.communicate()[0]
	
            numberList = re.findall('\.?\d+\.?\d*kb', output)
            lastSize = numberList[-1]
	
            lastSizeList = re.findall('\.?\d+\.?\d*', lastSize)
            saveSize = lastSizeList[0]
            if saveSize.startswith('.'):
                saveSize = '0' + saveSize
	
            finalSize = float(saveSize)
            all_file_size += finalSize
            all_file_count += 1
            print output
```

其他的一些减包方案就不展开了，接下来我试着分析一下 bestswifter 大神的 `BSBacktraceLogger `

<h3 id="获取调用堆栈">获取调用堆栈</h3>
说到调用堆栈，我们很容易联想到 DSYM 文件，我们知道 Xcode build setting 有个 DEBUG INFOMATION FORMAT 的选项

![][image-18]

可以看到 Debug 模式下，符号表文件会存入可执行文件中，而 Release 模式则会生成出 DSYM 文件，我们平常使用 Bugly 等工具上传的就是这份 DSYM 文件，DSYM 也是种 Mach-O 文件。在 Debug 模式，由于符号表在内存中，这为我们符号化堆栈提供了可能性。

```objc
bool bs_fillThreadStateIntoMachineContext(thread_t thread, _STRUCT_MCONTEXT *machineContext) {
    mach_msg_type_number_t state_count = BS_THREAD_STATE_COUNT;
    kern_return_t kr = thread_get_state(thread, BS_THREAD_STATE, (thread_state_t)&machineContext->__ss, &state_count);
    return (kr == KERN_SUCCESS);
}
```

`thread_get_state ` 函数获取线程执行状态（例如寄存器），传入 `_STRUCT_MCONTEXT ` 结构体，`_STRUCT_MCONTEXT ` 在不同的 cpu 架构会有所不同。

```objc
uintptr_t bs_mach_instructionAddress(mcontext_t const machineContext){
    return machineContext->__ss.BS_INSTRUCTION_ADDRESS;
}
	
const uintptr_t instructionAddress = bs_mach_instructionAddress(&machineContext);
```

获取当前指令的地址，也就是当前的栈帧，即当前被调用的函数。下面先讲下关于栈帧的概念。

#### 栈帧是什么

![][image-19]

如上图，一个函数调用栈是由若干个栈帧组成，每个栈帧通过 FP 和 SP 划分界线，fun1 函数 SP 和 FP 的指向就是 main 函数的栈帧。所以说只要知道当前函数的栈帧就能获取上一个函数的栈帧，从而回溯出函数调用栈。

程序计数器（PC）作用是给出将要执行的下一条指令在内存中的地址，上面代码的 `BS_INSTRUCTION_ADDRESS`。其中 16 位为 %ip，32 位为 %eip，64 位为 %rip，arm 是 pc。

SP 是栈指针寄存器，指向栈顶。

FP 是栈基址寄存器，指向栈起始位置。

LR 寄存器在子程序调用时会存储 PC 的值，即返回值。

为了方便获取栈帧，干脆构造一个栈帧的结构体，以下代码来自 KSCrash，它的注释已经很好的讲明了结构体的原由，BSBacktraceLogger 与之类似。

```objc
/** Represents an entry in a frame list.
 * This is modeled after the various i386/x64 frame walkers in the xnu source,
 * and seems to work fine in ARM as well. I haven't included the args pointer
 * since it's not needed in this context.
 */
typedef struct FrameEntry
{
    /** The previous frame in the list. */
    struct FrameEntry* previous;
	
    /** The instruction address. */
    uintptr_t return_address;
} FrameEntry;
```

之后，递归获取函数栈帧

```objc
for(; i < 50; i++) {
        backtraceBuffer[i] = frame.return_address;
        if(backtraceBuffer[i] == 0 ||
           frame.previous == 0 ||
           bs_mach_copyMem(frame.previous, &frame, sizeof(frame)) != KERN_SUCCESS) {
            break;
        }
    }
```

#### 符号化
符号化地址的大致思路分三步：1. 获取地址所在的内存镜像；2. 定位到内存镜像的符号表；3. 再从符号表中找到目标地址的符号。

##### 找到地址所在的内存镜像

```objc
uint32_t bs_imageIndexContainingAddress(const uintptr_t address) {
    const uint32_t imageCount = _dyld_image_count();
    const struct mach_header* header = 0;
	
    for(uint32_t iImg = 0; iImg < imageCount; iImg++) {
        header = _dyld_get_image_header(iImg);
```
        
遍历 image，得到指向 image header 的指针

```objc
uintptr_t addressWSlide = address - (uintptr_t)_dyld_get_image_vmaddr_slide(iImg);
uintptr_t cmdPtr = bs_firstCmdAfterHeader(header);
```

对指针 +1 操作，返回指向 load command 的指针

```objc
for(uint32_t iCmd = 0; iCmd < header->ncmds; iCmd++) {
                const struct load_command* loadCmd = (struct load_command*)cmdPtr;
                if(loadCmd->cmd == LC_SEGMENT) {
                    const struct segment_command* segCmd = (struct segment_command*)cmdPtr;
                    if(addressWSlide >= segCmd->vmaddr &&
                       addressWSlide < segCmd->vmaddr + segCmd->vmsize) {
                        return iImg;
                    }
                }
```

如果某个 segment 包含这个地址，那么该地址应大于 segment 的起始地址，小于 segment 的起始地址 + segment 的大小。

##### 定位镜像的符号表
`__LINKEDIT` 段包含了符号表（symbol），字符串表（string），重定位表（relocation）。`LC_SYMTAB` 指明了 `__LINKEDIT` 段查找字符串和符号表的位置。我们可以结合 `SEG_LINKEDIT` 和 `LC_SYMTAB` 来找到 image 的符号表。
接下来看看段基址的获取：
虚拟地址偏移量 = 虚拟地址（vmaddr） - 文件偏移量（fileoff）
段基址 = 虚拟地址偏移量 +  ASLR的偏移量

```objc 
const uintptr_t imageVMAddrSlide = (uintptr_t)_dyld_get_image_vmaddr_slide(idx);
// ALSR
const uintptr_t addressWithSlide = address - imageVMAddrSlide;
const uintptr_t segmentBase = bs_segmentBaseOfImageIndex(idx) + imageVMAddrSlide;
```
	
有了段基址，获取符号表和字符串表就只是计算下 symoff 和 stroff 偏移量了：

```objc
const BS_NLIST* symbolTable = (BS_NLIST*)(segmentBase + symtabCmd->symoff);
const uintptr_t stringTable = segmentBase + symtabCmd->stroff;
```

##### 找到最匹配的符号
递归查找离 addressWithSlide 更近的函数入口地址，因为 addressWithSlide 肯定大于某个函数的入口。

```objc
for(uint32_t iSym = 0; iSym < symtabCmd->nsyms; iSym++) {
                // If n_value is 0, the symbol refers to an external object.
    if(symbolTable[iSym].n_value != 0) {
            uintptr_t symbolBase = symbolTable[iSym].n_value;
                 uintptr_t currentDistance = addressWithSlide - symbolBase;
                  if((addressWithSlide >= symbolBase) &&
                       (currentDistance <= bestDistance)) {
                        bestMatch = symbolTable + iSym;
                        bestDistance = currentDistance;
                    }
        }
}
```

### 如何用 MachO 文件关联类的方法名

MachO 文件的 `__Text` 段有 `__objc_classname` 和 `__objc_methname` 来表示类名和方法名，但是这两者之间是如何做到关联的呢？下面我以系统的计算器做例子，试着进一步研究下 MachO 文件。
使用 MachOView 打开系统计算机，先来看看 `__objc_classname` 和 `__objc_methname` 在 load commands 里的定义：

![][image-20]

![][image-21]

我们顺着 `__objc_classname` 的偏移offset 109518 即 0x1ABCE 来到：

![][image-22]

同理 `__objc_methname` 的偏移为 0x165E8：

![][image-23]

那么，怎样像 class-dump 那样将类和自个的方法名对应起来呢？
由于每个类的虚拟地址都在Data 段 `__objc_classlist` 中：

![][image-24]

我们看到起始地址对应的是 0x1000298A8 这个地址，为了得到实际的地址需要用虚拟地址 - 段起始地址 + 文件偏移，经过一番计算，结果是0x298A8，来到文件偏移处，已经在DATA 段的 `__objc_data`

![][image-25]

在这里会对应着类的结构体，代码拷自 class-dump

```c
struct cd_objc2_class {
    uint64_t isa;
    uint64_t superclass;
    uint64_t cache;
    uint64_t vtable;
    uint64_t data; // points to class_ro_t
    uint64_t reserved1;
    uint64_t reserved2;
    uint64_t reserved3;
};
```

data 是我们感兴趣的，它指向 `class_ro_t `，熟悉 runtime 的话应该知道 `class_ro_t ` 存储了类在编译器就确定的属性、方法、协议等。
所以上图顺着找下去 0x100020A68 就是data 的内存地址，再用上面的公式计算得到 0x20A68，我们在 `__objc_const`找到那里：

![][image-26]

这里就是对应着 `class_ro_t `，来看看它在 class-dump 里的定义：

```c
struct cd_objc2_class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved; // *** this field does not exist in the 32-bit version ***
    uint64_t ivarLayout;
    uint64_t name;
    uint64_t baseMethods;
    uint64_t baseProtocols;
    uint64_t ivars;
    uint64_t weakIvarLayout;
    uint64_t baseProperties;
};
```
	
最终 0x20A80 就是name，0x20A88 就是 baseMethods。name 对应的正好是 0x1ABCE，类名是 BitFieldBox。baseMethods 指向内存 0x100020A00，该地址对应的数据是 18 00 00 00 04 00 00 00 表示 entsize 和 count 方法数，在这8个字节之后就是 name 方法名，types 方法类型， imp 函数指针了，所以方法名处的数据为 0x1000165e8 刚好对应 initWithFrame:
将结论用 class-dump 验证可得 BitFieldBox 的第一个方法是 initWithFrame

![][image-27]

### 总结
最初学习 MachO 文件格式觉得挺抽象的，后来经过各种源码的阅读和融合，终于在一次次地探索中比较直观地认识了 MachO 文件，特别是在 MachO 文件关联类的方法名时对类在内存中的布局有了更进一步的认识。虽然我们平常开发基本不和 MachO 文件打交道，但是对它有个基本概念，无论是做崩溃分析、逆向等都是有帮助的。

#### 参考链接
[深入剖析Macho (1)][3]

[iOS中线程Call Stack的捕获和解析（一）][4]

[iOS中线程Call Stack的捕获和解析（二）][5]

[获取任意线程调用栈的那些事][6]


[1]:	#%E5%87%8F%E5%B0%91%E5%8C%85%E5%A4%A7%E5%B0%8F
[2]:	#%E8%8E%B7%E5%8F%96%E8%B0%83%E7%94%A8%E5%A0%86%E6%A0%88
[3]:	http://satanwoo.github.io/2017/06/13/Macho-1/
[4]:	https://blog.csdn.net/jasonblog/article/details/49909163
[5]:	https://blog.csdn.net/jasonblog/article/details/49909209
[6]:	https://bestswifter.com/callstack/

[image-1]:	https://user-images.githubusercontent.com/5633917/37776837-23a2b58c-2e21-11e8-8b97-f968d484319e.png
[image-2]:	https://user-images.githubusercontent.com/5633917/37776371-15ac9b10-2e20-11e8-9dfe-850250c2a8bc.png
[image-3]:	https://user-images.githubusercontent.com/5633917/37776955-6ac166f2-2e21-11e8-9408-e7a3836eff72.png
[image-4]:	https://user-images.githubusercontent.com/5633917/37776995-7e897378-2e21-11e8-9c86-913ba5277b39.png
[image-5]:	https://user-images.githubusercontent.com/5633917/37777034-98bdef9e-2e21-11e8-900a-34108ae119fa.png
[image-6]:	https://user-images.githubusercontent.com/5633917/37777147-d5f832e8-2e21-11e8-8c69-8dba2d32a631.png
[image-7]:	https://user-images.githubusercontent.com/5633917/37777211-f9635d98-2e21-11e8-9c0e-66560b6b3197.png
[image-8]:	https://user-images.githubusercontent.com/5633917/37777286-1b07ebf8-2e22-11e8-8587-4b4897f9dbc7.png
[image-9]:	https://user-images.githubusercontent.com/5633917/37777312-2f77f7f4-2e22-11e8-8736-12ab00714504.png
[image-10]:	https://user-images.githubusercontent.com/5633917/37777338-4106908e-2e22-11e8-94df-3328b570226f.png
[image-11]:	https://user-images.githubusercontent.com/5633917/37777400-68692d3a-2e22-11e8-8d20-b42a4d88b86c.png
[image-12]:	https://user-images.githubusercontent.com/5633917/37777431-81613ff8-2e22-11e8-8ab5-ff4bbf8bffb0.png
[image-13]:	https://user-images.githubusercontent.com/5633917/37777470-96d8cd9c-2e22-11e8-8cea-cf107dbb7858.png
[image-14]:	https://user-images.githubusercontent.com/5633917/37777493-a96c4948-2e22-11e8-80f6-20bb35175ea5.png
[image-15]:	https://user-images.githubusercontent.com/5633917/37777894-940abbe2-2e23-11e8-9000-2d1800633d4b.png
[image-16]:	https://user-images.githubusercontent.com/5633917/37777951-b34195a8-2e23-11e8-93df-b659b770ba0f.png
[image-17]:	https://user-images.githubusercontent.com/5633917/37777997-d2a381fe-2e23-11e8-9ec0-3c6ab5e20447.png
[image-18]:	https://user-images.githubusercontent.com/5633917/37778243-6492e91a-2e24-11e8-8fde-1ec051ffae0e.png
[image-19]:	https://user-images.githubusercontent.com/5633917/37778363-bd21bbf6-2e24-11e8-8545-4acaf084d954.png
[image-20]:	https://user-images.githubusercontent.com/5633917/37772635-327d2624-2e16-11e8-98fb-b5cea70f5154.png
[image-21]:	https://user-images.githubusercontent.com/5633917/37772656-4669a46e-2e16-11e8-832f-6351a28979e0.png
[image-22]:	https://user-images.githubusercontent.com/5633917/37772457-a6329546-2e15-11e8-807e-cbef3da6cbee.png
[image-23]:	https://user-images.githubusercontent.com/5633917/37772866-f014900a-2e16-11e8-8ee2-04793df6d995.png
[image-24]:	https://user-images.githubusercontent.com/5633917/37773001-5c068688-2e17-11e8-8ca6-defa18ac7428.png
[image-25]:	https://user-images.githubusercontent.com/5633917/37773272-210903b6-2e18-11e8-8d75-5612f5c24dbf.png
[image-26]:	https://user-images.githubusercontent.com/5633917/37773857-aca7264a-2e19-11e8-848f-e40a00038f7f.png
[image-27]:	https://user-images.githubusercontent.com/5633917/37774620-b04a4212-2e1b-11e8-9e45-22098920b939.png

