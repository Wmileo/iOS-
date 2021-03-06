**说一下对 `isa` 指针的理解， 对象的`isa` 指针指向哪里？`isa` 指针有哪两种类型？**

`isa` 等价于 `is kind of`

- 实例对象 `isa` 指向类对象
- 类对象指 `isa` 向元类对象
- 元类对象的 `isa` 指向元类的基类


`isa` 有两种类型
- 纯指针，指向内存地址
- `NON_POINTER_ISA`，除了内存地址，还存有一些其他信息


# isa源码分析

在Runtime源码查看isa_t是共用体。简化结构如下：

```objc
union isa_t 
{
    Class cls;
    uintptr_t bits;
    # if __arm64__ // arm64架构
#   define ISA_MASK        0x0000000ffffffff8ULL //用来取出33位内存地址使用（&）操作
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1; //0：代表普通指针，1：表示优化过的，可以存储更多信息。
        uintptr_t has_assoc         : 1; //是否设置过关联对象。如果没设置过，释放会更快
        uintptr_t has_cxx_dtor      : 1; //是否有C++的析构函数
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000 内存地址值
        uintptr_t magic             : 6; //用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1; //是否有被弱引用指向过
        uintptr_t deallocating      : 1; //是否正在释放
        uintptr_t has_sidetable_rc  : 1; //引用计数器是否过大无法存储在ISA中。如果为1，那么引用计数会存储在一个叫做SideTable的类的属性中
        uintptr_t extra_rc          : 19; //里面存储的值是引用计数器减1

#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__ // arm86架构,模拟器是arm86
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };

# else
#   error unknown architecture for packed isa
# endif

}
```

> 注意：什么是位域？