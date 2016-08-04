#Android N混合编译与对热补丁影响深度解析#
> 首先非常抱歉Tinker没有按期内测，这主要因为开源的代码需要通过公司内部审核与评测，这项工作大约还需要一个月左右。当前Tinker已经在公司内部开源，我们会努力让它以更完善的姿态与大家见面。

大约在六月底，Tinker在微信全量上线了一个补丁版本，随即华为反馈在Android N上微信无法启动。冷汗冒一地，Android N又搞了什么东东？为什么与instant run保持一致的补丁方式也跪了？talk is cheap，show me the code。趁着台风妮妲肆虐广东，终于有时间总结一把。在此非常感谢华为工程师谢小灵与胡海亮的帮助，事实上微信与各大厂商都保持着非常紧密的联系。

##无法启动的原因
我们遵循从问题出发的思路，针对华为提供的日志，我们看到微信在Android N上启动时会报`IllegalAccessError`。可以从`/data/user/0/com.tencent.mm/tinker/patch-a002c56d/dex/classes2.dex`看到，的确跟补丁是有关系的。

```
java.lang.IllegalAccessError: Illegal class access: 'com.tencent.mm.ui.conversation.ConversationOverscrollListView' attempting to access 'com.tencent.mm.ui.conversation.ConversationOverscrollListView$c' (declaration of 'com.tencent.mm.ui.conversation.ConversationOverscrollListView' appears in /data/user/0/com.tencent.mm/tinker/patch-a002c56d/dex/classes2.dex)
```

但是在我们手上Android N却无法复现，同时跟华为的进一步沟通中，他们也明确只有一少部分N的用户会出现问题。这就很难办了，但是根据之前在art地址错乱的经验(似乎这里我还欠大家一篇分析文章)，跟这里似乎有点相似。

但是Tinker已经做了全量替换，所以我怀疑由于Android N的某种机制这里只有部分用了补丁中的类，但是部分类导致使用了原来的dex中的。接下来就跟着我一起去研究Android N在编译运行究竟做了什么改变吧?

##Android N的混合编译运行模式
网上关于Android N混合编译的文章并不多，infoq上有一篇翻译文章：[Android N混合使用AOT编译，解释和JIT三种运行时](http://www.infoq.com/cn/news/2016/04/android-n-aot-jit)。混合编译运行主要指AOT编译，解释执行与JIT编译，它主要解决的问题有以下几个：

1. `应用安装时间过长`；在N之前，应用在安装时需要对所有ClassN.dex做AOT机器码编译，类似微信这种比较大型的APP可能会耗时数分钟。但是往往我们只会使用一个应用20%的功能，剩下的80%我们付出了时间成本，却没带来太大的收益。
2. `降低占ROM空间`；同样全量编译AOT机器码，12M的dex编译结果往往可以达到50M之多。只编译用户用到或常用的20%功能，这对于存储空间不足的设备尤其重要。
3. `提升系统与应用性能`；减少了全量编译，降低了系统的耗电。在boot.art的基础上，每个应用增加了base.art(这块后面会详细解析), 通过预加载与缓存提升应用性能。
4. `快速的系统升级`；以往厂商ota时，需要对安装的所有应用做全量的AOT编译，这耗时非常久。事实上，同样只有20%的应用是我们经常使用的，给不常用的应用，不常用的功能付出的这些成本是不值得的。

Android N为了解决这些问题，通过管理**解释，AOT与JIT**三种模式，以达到一种运行效率、内存与耗电的折中。简单来说，在应用运行时分析运行过的代码以及“热代码”，并将配置存储下来。在设备空闲与充电时，ART仅仅编译这份配置中的“热代码”。我们先来看看Android N上有哪些编译方法：

###Android N的编译模式
在[compiler_filter.h](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/compiler_filter.h#32)，我们可以看到dex2oat一共有12种编译模式：

```
enum Filter {   
    VerifyNone,           // Skip verification but mark all classes as verified anyway.
    kVerifyAtRuntime,     // Delay verication to runtime, do not compile anything.
    kVerifyProfile,       // Verify only the classes in the profile, compile only JNI stubs.
    kInterpretOnly,       // Verify everything, compile only JNI stubs.
    kTime,                // Compile methods, but minimize compilation time.
    kSpaceProfile,        // Maximize space savings based on profile.
    kSpace,               // Maximize space savings.
    kBalanced,            // Good performance return on compilation investment.
    kSpeedProfile,        // Maximize runtime performance based on profile.
    kSpeed,               // Maximize runtime performance.
    kEverythingProfile,   // Compile everything capable of being compiled based on profile.
    kEverything,          // Compile everything capable of being compiled.
};
```

以上12种编译模式**按照排列次序逐渐增强**，那系统默认采用了哪些编译模式呢？我们可以在在手机上执行`getprop | grep pm`查看:

```
pm.dexopt.ab-ota: [speed-profile]
pm.dexopt.bg-dexopt: [speed-profile]
pm.dexopt.boot: [verify-profile]
pm.dexopt.core-app: [speed]
pm.dexopt.first-boot: [interpret-only]
pm.dexopt.forced-dexopt: [speed]
pm.dexopt.install: [interpret-only]
pm.dexopt.nsys-library: [speed]
pm.dexopt.shared-apk: [speed]
```
其中有几个我们是特别关心的，

1. `install`(应用安装)与`first-boot`(应用首次启动)使用的是[interpret-only]，即只verify，代码解释执行即不编译任何的机器码，它的性能与Dalvik时完全一致，先让用户愉快的玩耍起来。
2. `ab-ota`(系统升级)与`bg-dexopt`(后台编译)使用的是[speed-profile]，即只根据“热代码”的profile配置来编译。这也是N中混合编译的核心模式。
3. 对于动态加载的代码，即`forced-dexopt`，它采用的是[speed]模式，即最大限度的编译机器码，它的表现与之前的AOT编译一致。

总的来说，程序使用loaddex动态加载的代码是无法享受混合编译带来的好处，我们应当尽量**采用ClassN.dex方式来符合Google的规范**。这不仅在ota还是混合编译上，都会带来很大的提升。

###Android N的Profile文件
在讲[speed-profile]是怎样编译之前，这里先简单描述一下profile文件。profile相关的核心代码都在art/runtime/jit中。简单来说，[profile_saver.cc](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/jit/profile\_saver.cc)会开启线程去专门收集已经resolved的类与函数，达到一定条件即会持久化存储在`/data/misc/profiles`文件夹中。具体的条件可以在[profile\\_saver\\_options.h](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/jit/profile_saver_options.h)中查看，在收集过程会出现类似以下的日志：

```
tinker.sample.android I/art: Collecting resolved classes
tinker.sample.android I/art: Collecting class profile for dex file /data/app/tinker.sample.android-1/base.apk types=2406 class_defs=1719
tinker.sample.android I/art: Dex location /data/app/tinker.sample.android-1/base.apk has 232 / 1719 resolved classes
```
profile的存储格式在[offline\\_profiling\\_info.h](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/jit/offline_profiling_info.h)中定义，我们也可以通过`profman`命令查看profile文件中的数据，命令如下：

```
profman --profile-file=/data/misc/profiles/cur/0/tinker.sample.android/primary.prof --dump-only
```

具体输出如下：

```
=== profile ===
ProfileInfo:
base.apk
methods: 297,302,303,424,427,665,668,669,700,756,757,759,760,761,765,766,768,772,774,
classes: 52,124,456,
```

其中base.apk代表dex的位置，这里代表的是ClassN中的第一个dex。其他dex会使用类似base.apk:classes2.dex方式命名。后面的methods与classes代表的是它们在dex格式中的index，只有这些类与方法是我们需要在[spped-profile]模式中需要编译。

###Android N的dex2oat编译
在这里我们比较关心系统究竟是什么时候会去对应用做类似增量的编译，还有具体的编译流程是怎么样的？

####dex2oat编译的时机
首先我们来看系统在什么时候会对各个应用做这种渐进式编译呢？手机在充电＋空闲＋四个小时间隔等多个条件下，通过[BackgroundDexOptService.java](https://android.googlesource.com/platform/frameworks/base/+/android-n-preview-5/services/core/java/com/android/server/pm/BackgroundDexOptService.java)中的JobSchedule下触发编译优化。

```
new JobInfo.Builder(BACKGROUND_DEXOPT_JOB, sDexoptServiceName)
           .setRequiresDeviceIdle(true)
           .setRequiresCharging(true)
           .setMinimumLatency(minLatency)
           .build();
```
####dex2oat编译的流程
对于[speed-profile]模式，dex2oat编译命令的核心参数如下：

```
dex2oat --dex-file=./base.apk --oat-file=./base.odex --compiler-filter=speed-profile --app-image-file=./base.art
 --profile-file=./primary.prof ...
```

入口文件位于[dex2oat.cc](https://android.googlesource.com/platform/art/+/android-n-preview-5/dex2oat/dex2oat.cc)中，在这里并不想贴具体的调用函数，简单的描述一下流程：若dex2oat参数中有输入profile文件，会读取profile中的数据。与以往不同的是，这里不仅会根据profile文件来生成base.odex文件，同时还会生成称为app_image的base.art文件。与boot.art类似，base.art文件主要为了加快应用的对“热代码”的加载与缓存。

我们可以通过oatdump命令来看到art文件的内容，具体命令如下：

```
oatdump --app-image=base.art --app-oat=base.odex  --image=/system/framework/boot.art  --instruction-set=arm64
```
我们可以dump到art文件中的所有信息，这里我只将它的头部信息输出如下：

```
IMAGE LOCATION: base.art
IMAGE BEGIN: 0x77ea1000
IMAGE SIZE: 1597200
IMAGE SECTION SectionObjects: size=2040 range=0-2040
IMAGE SECTION SectionArtFields: size=0 range=2040-2040
IMAGE SECTION SectionArtMethods: size=0 range=2040-2040
IMAGE SECTION SectionRuntimeMethods: size=0 range=2040-2040
IMAGE SECTION SectionIMTConflictTables: size=0 range=2040-2040
IMAGE SECTION SectionDexCacheArrays: size=1591080 range=2040-1593120
IMAGE SECTION SectionInternedStrings: size=4040 range=1593120-1597160
IMAGE SECTION SectionClassTable: size=40 range=1597160-1597200
IMAGE SECTION SectionImageBitmap: size=4096 range=1597440-1601536
```

base.art文件主要记录已经编译好的类的具体信息以及函数在oat文件的位置，一个class的输出格式如下：

```
0x78c8f768: java.lang.Class "com.tencent.mm.ui.d.a" (StatusInitialized)
    shadow$_klass_: 0x6fc76488   Class: java.lang.Class
    shadow$_monitor_: 0 (0x0)
    accessFlags: 524305 (0x80011)
    annotationType: null   sun.reflect.annotation.AnnotationType
    classFlags: 0 (0x0)
    classLoader: 0x787b5140   java.lang.ClassLoader
    classSize: 460 (0x1cc)
    clinitThreadId: 0 (0x0)
    componentType: null   java.lang.Class
    copiedMethodsOffset: 3 (0x3)
    dexCache: 0x782290c8   java.lang.DexCache
    dexCacheStrings: 2036372056 (0x79609258)
    dexClassDefIndex: 12138 (0x2f6a)
    dexTypeIndex: 11797 (0x2e15)
    iFields: 2031076964 (0x790fc664)
    ifTable: 0x78836500   java.lang.Object[]
    methods: 2032787876 (0x7929e1a4)
    name: null   java.lang.String
    numReferenceInstanceFields: 4 (0x4)
    numReferenceStaticFields: 0 (0x0)
    objectSize: 36 (0x24)
    primitiveType: 131072 (0x20000)
    referenceInstanceOffsets: 63 (0x3f)
    sFields: 0 (0x0)
    status: 10 (0xa)
    superClass: 0x78bcc968   Class: com.tencent.mm.pluginsdk.ui.b.b
    verifyError: null   java.lang.Object
    virtualMethodsOffset: 1 (0x1)
    vtable: null   java.lang.Object
```

method的输出格式如下：

```
0x792b639c  ArtMethod: void com.tencent.mm.e.a.je.<init>()
OAT CODE: 0x471dae14-0x471daece
SIZE: Dex Instructions=10 StackMaps=0 AccessFlags=0x90001
0x792b63c0  ArtMethod: void com.tencent.mm.e.a.je.<init>(byte)
OAT CODE: 0x471daee4-0x471daf52
SIZE: Dex Instructions=48 StackMaps=0 AccessFlags=0x90002  
0x792b63e8  ArtMethod: void com.tencent.mm.e.a.jo.<init>()
OAT CODE: 0x463d5f44-0x463d5f50
SIZE: Dex Instructions=10 StackMaps=0 AccessFlags=0x90001
```

那么我们就剩下最后一个问题，app image文件是什么时候被加载，并且为什么它会影响热补丁的机制？

###App image文件的加载
在apk启动时我们需要加载应用的oat文件以及可能存在的app image文件，它的大致流程如下：

1. 通过[OpenDexFilesFromOat](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/oat_file_manager.cc#541)加载oat时，若app image存在，则通过调用[OpenImageSpace](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/oat_file_assistant.cc#989)函数加载；
2. 在加载app image文件时，通过[UpdateAppImageClassLoadersAndDexCaches](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/class_linker.cc#1227)函数，将art文件中的dex\_cache中dex的所有class插入到ClassTable，同时将method更新到dex\_cache;
3. 在类加载时，使用时[ClassLinker::LookupClass](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/class_linker.cc#3599)会先从ClassTable中去查找，找不到时才会走到[DefineClass](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/class_linker.cc#2437)中。

非常简单的说，app image的作用是记录已经编译好的“热代码”，并且在启动时一次性把它们加载到缓存。预先加载代替用时查找以提升应用的性能，到这里我们终于明白为什么base.art会影响热补丁的机制。

**无论是使用插入pathlist还是parent classloader的方式，若补丁修改的class已经存在与app image，它们都是无法通过热补丁更新的。它们在启动app时已经加入到PathClassLoader的ClassTable中，系统在查找类时会直接使用base.apk中的class。**

###instant run为什么没有影响
对于instant run来说，它的目标是快速debug。从上面的编译条件看来，它是不太可能可以触发[speed-profile]编译的。事实上，它在dex2oat上面传入了**--debugable**参数， 不过dex2oat并没有单独处理这个参数。感兴趣的同学，可以再详细研究这一块。

最后我们再来总结一下Android N混合编译运行的整个流程，它就像一个小型生态系统那样和谐。

![](http://i.imgur.com/aT7JJJs.png)

##Android N上热补丁的出路
假设base.art文件在补丁前已经存在，这里存在三种情况：

1. 补丁修改的类都不app image中；这种情况是最理想的，此时补丁机制依然有效；
2. 补丁修改的类部分在app image中；这种情况我们只能更新一部分的类，此时是最危险的。一部分类是新的，一部分类是旧的，app可能会出现地址错乱而出现crash。
3. 补丁修改的类全部在app image中；这种情况只是造成补丁不生效，app并不会因此造成crash。

如何解决这个问题呢？下面根据当时我的一些思路分别说明：

###插桩？
当时第一反应想到是通过插桩是否能阻止类被编译到app image中，从而规避了这个问题。事实上，在生成profile时，使用的是[ClassLinker::GetResolvedClasses](https://android.googlesource.com/platform/art/+/android-n-preview-5/runtime/class_linker.cc#8111)函数，插桩并没有任何作用。

我这边也专门单独看了插桩后编译的机器码，仅仅是通过Trampoline模式跳回虚拟机查找而已。

```
 DEX CODE:
     ...
     0x0018: 0e00                         | return-void
     0x45f0dda2: f8d9e29c    ldr.w   lr, [r9, #668]  ; pInvokeStaticTrampolineWithAccessCheck
     ...
```

###miniloader方案
假设我们实现一个最小化的loader,这部分代码我们补丁时是不会去改变。然后其他代码都通过动态方式加载，这套方案的确是可行的，但是并不会被采用，因为它会带来以下几个代价：

1. 对Android N之前，由于不使用ClassN方式，带来首次加载过慢甚至黑屏的问题；
2. 对于Android N，不仅存在第一点问题，同时将混合编译的好处完全废掉了(因为动态加载的代码是相当于完全编译的)；

在微信中，补丁方案的原则应该是不能影响运行时的性能，所以这套方案也是不可取的。

###运行时替换PathClassLoader方案
事实上，App image中的class是插入到PathClassloader中的ClassTable中。假设我们完全废弃掉PathClassloader，而采用一个新建Classloader来加载后续的所有类，即可达到将cache无用化的效果。

需要注意的问题是我们的Application类是一定会通过PathClassloader加载的，所以我们需要将Application类与我们的逻辑解耦，这里方式有两种：

1. 采用类似instant run的实现；在代理application中，反射替换真正的application。这种方式的优点在于接入容易，但是这种方式无法保证兼容性，特别在反射失败的情况，是无法回退的。
2. 采用代理Application实现的方法；即Application的所有实现都会被代理到其他类，Application类不会再被使用到。这种方式没有兼容性的问题，但是会带来一定的接入成本。

我想说明的是许多号称毫无兼容性问题的反射框架，在微信Android 数亿用户面前往往都是经不起考验的。这也是为什么我们尽管采用增加接入成本方式也不愿意再多的使用反射的原因。总的来说，这种方式不会影响没有补丁时的性能，但在加载补丁后，由于废弃了App image带来一定的性能损耗。具体数据如下：

![](http://i.imgur.com/Z42vvMJ.png)

事实上，在Android N上我们不会出现完整编译一个应用的base.odex与base.art的情况。base.art的作用是加快类与方法的第一次查找速度，所以在启动时这个数据是影响最大的。在这种情况，废弃base.art大约带来15%左右的性能损耗。在其他情况下，这个数字应该是远远小于这个数字。

##Tinker的后续计划
在Android N上，Tinker全量合成方案带来了一个较为严重的问题。即将Android N的混合编译退化了，因为动态编译的代码采用的是[speed]方式完整编译，它会占用比较多Rom空间。所以未来我们计划根据平台区分合成的方式，在Dalvik平台我们合成一个完整的dex，但在Art平台只合成需要的类，它的规则如下：

1. 修改跟新增的class；
2. 若class有field，method或interface数量变化，它们所有的子类；
3. 若class有field，method或interface数量变化d，它们以及它们所有子类的调用类。如果采用ClassN方式，即需要多个dex一起处理。

规则看起来很复杂，同一个diff文件，根据不同平台合成不同文件看起来也很复杂。更困难的是，dex格式是存在大量的互相引用，除了index区域，还有使用绝对地址引用的区域，大量的变长结构，4字节对齐......

所以Tinker最终期望的结构图应该如下，在art上面仅仅合成mini.dex即可：
![](http://i.imgur.com/6OPWAYj.png)

##结语
建议大家通过"阅读全文"查看，以获得更好的阅读体验。尽管当前Tinker还没有开启内测，我们会尽力在开源前做的更好。让Tinker无论在Dalvik还是Art上，都有着最好的表现，同时也恳请大家继续耐心等候我们。



