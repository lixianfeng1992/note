Objective-C 2.0 中 `isa` 指针被 `isa_t`代替, 这个结构体中”包含”了当前对象指向的类的信息
```c
struct objc_object {
  isa_t isa;
}
```
```c
struct objc_class: objc_object {
  isa_t isa;
  Class superclass;
  cache_t cache;
  class_data_bits_t bits;
}
```
对象isa -> class</br>

class isa -> meta class</br>

当实例方法被调用时，它要通过自己持有的 isa 来查找对应的类，然后在这里的 class_data_bits_t 结构体中查找对应方法的实现。同时，每一个 objc_class 也有一个指向自己的父类的指针 super_class 用来查找继承的方法。</br>

```c
#define ISA_MASK        0x00007ffffffffff8ULL
#define ISA_MAGIC_MASK  0x001f800000000001ULL
#define ISA_MAGIC_VALUE 0x001d800000000001ULL
#define RC_ONE   (1ULL<<56)
#define RC_HALF  (1ULL<<7)

union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44;
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
    };
};
```

## isa的初始化
```c
inline void
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor) {
  initIsa(cls, true, hasCxxDtor);
}
```
```c
objc_object::initIsa(Class cls, bool indexed, bool hasCxxDtor) {
  if (!indexed) {
    isa.cls = cls;
  } else {
    isa.bits = ISA_MAGIC_VALUE;
    isa.has_cxx_dtor = hasCxxDtor;
    isa.shiftcls = (uintptr_t)cls >> 3
  }
}
```
```c
#define FAST_IS_SWIFT           (1UL<<0)
#define FAST_HAS_DEFAULT_RR     (1UL<<1)
#define FAST_REQUIRES_RAW_ISA   (1UL<<2)
#define FAST_DATA_MASK          0x00007ffffffffff8UL

struct class_data_bits_t {
  uintptr_t bits;
public:
  class_rw_t* data() {
      return (class_rw_t *)(bits & FAST_DATA_MASK);
  }
}
```
```c
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;
};
```
`class_ro_t` 结构体中存储当前类在编译期就已经确定的属性、方法以及遵循的协议。
```c
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;

    const uint8_t * ivarLayout;

    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};
```
在编译期间类的结构中的 `class_data_bits_t *data` 指向的是一个 `class_ro_t *` 指针：</br>
然后在加载 ObjC 运行时的过程中在 realizeClass 方法中：
```c
const class_ro_t *ro = (const class_ro_t *)cls->data();
class_rw_t *rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
rw->ro = ro;
rw->flags = RW_REALIZED|RW_REALIZING;
cls->setData(rw)
```

## 方法的结构
```c
struct method_t {
    SEL name;
    const char *types;
    IMP imp;
};

在控制台中的输出:
name = "hello"
types = 0x0000000100000fa4 "v16@0:8"
imp = 0x0000000100000e90 (method`-[XXObject hello] at XXObject.m:13
```

# 从源代码看 ObjC 中消息的发送
```
[object hello] -> objc_msgSend(object, @selector(hello))
```
`@selector(hello)`的作用是生成选择子`SEL`

##　解析 objc_msgSend
```
0 lookUpImpOrForward
1 _class_lookupMethodAndLoadCache3
2 objc_msgSend
3 main
4 start
```
调用栈在这里告诉我们： lookUpImpOrForward 并不是 objc_msgSend 直接调用的，而是通过 _class_lookupMethodAndLoadCache3 方法：
```c
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    return lookUpImpOrForward(cls, sel, obj,
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
```
## 实现的查找 lookUpImpOrForward
由于实现的查找方法 lookUpImpOrForward 涉及很多函数的调用，所以我们将它分成以下几个部分来分析：
1. 无锁的缓存查找
2. 如果类没有实现（isRealized）或者初始化（isInitialized），实现或者初始化类
3. 加锁
4. 缓存以及当前类中方法的查找
5. 尝试查找父类的缓存以及方法列表
6. 没有找到实现，尝试方法解析器
7. 进行消息转发
8. 解锁、返回实现

### 无锁的缓存查找
下面是在没有加锁的时候对缓存进行查找，提高缓存使用的性能：
```c
runtimeLock.assertUnlocked();

// Optimistic cache lookup
if (cache) {
   imp = cache_getImp(cls, sel);
   if (imp) return imp;
}
```
不过因为 _class_lookupMethodAndLoadCache3 传入的 cache = NO，所以这里会直接跳过 if 中代码的执行，在  objc_msgSend 中已经使用汇编代码查找过了。

### 类的实现和初始化
在 Objective-C 运行时 初始化的过程中会对其中的类进行第一次初始化也就是执行 realizeClass 方法，为类分配可读写结构体 class_rw_t 的空间，并返回正确的类结构体。
而 _class_initialize 方法会调用类的 initialize 方法，我会在之后的文章中对类的初始化进行分析。
```c
if (!cls->isRealized()) {
    rwlock_writer_t lock(runtimeLock);
    realizeClass(cls);
}

if (initialize  &&  !cls->isInitialized()) {
    _class_initialize (_class_getNonMetaClass(cls, inst));
}
```
### 加锁
加锁这一部分只有一行简单的代码，其主要目的保证方法查找以及缓存填充（cache-fill）的原子性，保证在运行以下代码时不会有新方法添加导致缓存被冲洗（flush）。
```
runtimeLock.read();
```

### 在当前类中查找实现
```c
imp = cache_getImp(cls, sel);
if (imp) goto done;
```

## 方法决议
选择子在当前类和父类中都没有找到实现，就进入了方法决议（method resolve）的过程：
```c
if (resolver  &&  !triedResolver) {
    _class_resolveMethod(cls, sel, inst);
    triedResolver = YES;
    goto retry;
}
```
```c
void _class_resolveMethod(Class cls, SEL sel, id inst) {
  if (!cls->isMetaClass()) {
    _class_resolveInstanceMethod(cls, sel, inst);
  } else {
    _class_resolveClassMethod(cls, sel, inst);
    if (!lookUpImpOrNil(cls, sel, inst,
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)))
    {
      _class_resolveInstanceMethod(cls, sel, inst);
    }
  }
}
```
```c
static void _class_resolveInstanceMethod(Class cls, SEL sel, id inst) {
  if (!lookUpImpOrNil(cls->ISA(), SEL_resolveInstanceMethod, cls,
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) {
    // 没有找到 resolveInstanceMethod: 方法，直接返回。
    return;
  }

  BOOL (*msg)(Class, SEL, SEL) = (__typeof__(msg))objc_msgSend;
  bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

  // 缓存结果，以防止下次在调用 resolveInstanceMethod: 方法影响性能。
  IMP imp = lookUpImpOrNil(cls, sel, inst,
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);
}
```

## 消息转发
在缓存、当前类、父类以及 resolveInstanceMethod: 都没有解决实现查找的问题时，Objective-C 还为我们提供了最后一次翻身的机会，进行方法转发：
```
imp = (IMP)_objc_msgForward_impcache;
cache_fill(cls, sel, imp, inst);
```
返回实现 _objc_msgForward_impcache，然后加入缓存。
