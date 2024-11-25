---
title: "C#中的指针/不安全代码"
published: 2022-12-28T18:38:08+08:00
draft: false
---



## 基本知识

在C#中, 也是可以使用指针的, 它的指针和C语言类似, 只不过要开启unsafe(不安全代码)  
一般来说, 不安全代码就是使用了指针的代码  

### 启用unsafe

在项目属性中, 修改设置允许unsafe, 或者编辑项目.csproj, 在PropertyGroup里将AllowUnsafeBlocks设置为True  
在.net framework中, csproj可能有些不一样

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net6.0</TargetFramework>
    <Nullable>enable</Nullable>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>
</Project>
```

### unsafe范围

需要项目设置为允许unsafe, 还需要使用unsafe代码块才能使用指针  
可以把class,struct,method,标注为unsafe, 之后在里面使用指针而无需在每一个地方都使用unsafe

```csharp
class Foo
{
    public unsafe int* Ptr;
    public void Hi()
    {
        int a = 10;
        unsafe
        {
            int*p = &a;
            *p = 30;
        }
    }
}
```

```csharp
unsafe class Foo //可以在Foo里面随意使用不安全代码而无需再次标注unsafe
{
    public int* Ptr;
    public void Hi()
    {
        int a = 10;
        int* p = &a;
        *p = 30;
    }
}
```

### 托管类型的指针

**在Visual Studio 2022 v17.4+中, 指针可以是托管类型的指针**  
注意, 这里是受限于IDE, 而不是C#语言本身, 可能是加了编译器参数(具体不清楚)  
也就是说, 如果你使用了vs 17.4+, 即使你用的.net framework C#7.0 也可以使用托管类型的指针

```csharp
string str = "abc";

//但是注意, 这里是获取了str这个变量在堆栈上的地址, 并不是str引用的地址, 也不是储存字符串的字符的地址
string* p = &str; //在vs v17.4以前,是不允许的
object* a;
Func<string>* b; //这不是函数指针
IEnumerable<char>* c; //这里不能 c = &str;  这个c是指向一个接口变量,它是指向变量的地址的
int?* d;

int[]* e; //这个不能在C#中显式声明,会报错, 但是存在有T[]*这种类型的
string[] strs = { "a", "b", "c" };
var sp = &strs; //此时它的类型是string[]* ,这里只能使用var
```

### 栈和堆, 值类型和引用类型

~~stack译作堆(叠) 也就是栈, heap是堆, 而the stack又被翻译成堆栈, 翻译有些不恰当~~  
在整篇文章中, 将使用`栈`来表示 Microsoft文档中所说`堆栈`和`stack`,  `堆`来表示`heap`

在方法中, 变量是声明在栈上的, 你的所有变量都在栈上\*  
定义引用类型的变量, 先会new在堆中, 然后在栈上储存对象的引用(地址)  
也就是说,对象本身在堆中, 栈上储存了它们的地址

```csharp
int num = 20; //在栈上储存着int的值(4字节)
object obj = new Foo(); //在栈上储存着对象的引用(8字节)
Foo2 foo2 = new(); //在栈上储存着foo2这个结构体(16字节)

class Foo
{
    public string Name;
    public long Value;
}
struct Foo2
{
    public string Name;
    public long Value;
}
```

#### 值类型

值类型声明在栈上, 它只储存值, 不储存对象头, 但是运行的时候是怎么知道它是什么类型的呢?  
在声明方法, 会把这个信息写入到方法信息中(应该是元数据), 在声明类的时候, 会把这个类型信息写入到class的元数据中, 它在编译后就是固定的, 类型信息和大小不会变化  

所以结构体也不能继承, 只能从ValueType派生过来, 不能再次派生  
在对结构体使用接口的时候, 会被装箱, 装箱后它的性质就和class几乎一样了, 也有对象头, 会把数据拷贝一份到堆中并加一个对象头和SyncBlock  
这样就可以确认这个接口实现的类型是什么了, 不然被调用结构里的方法, 它不知道调用哪个具体实现的类型方法

```msil
; .il method的一部分定义
.locals (
    [0] int32 ; int类型的变量
    )
```

#### 引用类型在堆中的结构

如果在栈上声明的引用变量,它指向的就是这个Header(MethodTable\*)的起始位置  
可以通过`typeof(string).TypeHandle.Value`来获取对象头(MethodTable\*)  
一个引用类型储存在堆中, 它就是依靠对象头来确定对象的类型的

引用类型在堆中储存(64位下):

| SyncBlock | Header(MethodTable\*) | Data(数据区域)      |
| --------- | --------------------- | ------------------- |
| 8字节     | 8字节                 | 成员(至少占用8字节) |

一个object它即使没有成员, 数据区域也会占用8字节, 也就是说, 一个Object, 它在内存中最少都有24字节  
在MethodTable中, BaseSize指定了一个对象在内存中需要申请/占用的字节数  
但是String和Array例外, 它们有额外的大小, BaseSize+成员大小=它们实际大小

假设以上面的例子class Foo ,   整个对象就占用32字节

| SyncBlock | Header(MethodTable\*) | string Name | long Value |
| --------- | --------------------- | ----------- | ---------- |
| 8字节     | 8字节                 | 8字节       | 8字节      |

再以字符串"abcd"为例, string在堆中储存是有\0结尾的, 这个字符串的成员占用14字节, SyncBlock是8字节,头是8字节, 一共30字节

| SyncBlock(8字节) | Header(8字节)                   | Length(4字节) | chars(8字节)                        | end(2字节) |
| ---------------- | ------------------------------- | ------------- | ----------------------------------- | ---------- |
| 8字节            | typeof(string).TypeHandle.Value | 4             | "abcd"                              | \0         |
| 0                | 0x00007ff?????????              | 0x00000004    | `0x0061` `0x0062` `0x0063` `0x0064` | 0x0000     |

#### 类型的成员顺序

struct和class都有一个`StructLayoutAttribute`特性, 它来反应类型在内存里是如何排列布局的, 通常只要关心它的`Value(LayoutKind)`属性  
StructLayout特性它是隐式的, 你可以标注此特性覆盖默认值

- Sequential  成员在内存中是按顺序储存的
- Auto  成员会自动选择适当的布局, 它们在内存中的顺序可能会发生改变
- Explicit  显式控制成员在内存中的顺序

struct 它的StructLayout.Value默认是LayoutKind.Sequential  
class 它的StructLayout.Value默认是LayoutKind.Auto

#### 结构体的内存对齐

它会根据`StructLayout.Pack`来控制内存对齐, Pack表示最大字节对齐是多少  
默认Pack为0, 如果Pack=0, 则不受最大限制, 按照按照结构体最大成员大小来指定对齐  
只有LayoutKind.Sequential才会受Pack的影响

Int128、UInt128、Vector128、Vector256和Vector512是特殊的  
它们是唯一大于8字节的Pack类型, 当包含它们时, 会以它们的大小来进行内存对齐, 而非8字节
例子: 虽然Int128是以两个ulong组成的, 但是它特殊, 不会按照8字节对齐, 而是16字节, 请看Foo4

如果声明了int类型的字段, 则它会以4字节对齐  
如果声明了long类型的字段, 则会以8字节对齐, 但是最大不会超过Pack的大小  
以下是在x64下定义

```csharp
//这里面最大的成员是int, 所以最大以4字节对齐
struct Foo1 // 12字节
{
    public byte n1; //1字节
    //内存对齐 空的3字节
    public int n2; //4字节   声明过int, 所以最大以4字节对齐
    public int n3; //4字节
}

struct Foo2 //24字节
{
    public int n1; //4字节
    // 内存对齐 空的4字节
    public long n2; // 8字节   声明过long, 所以最大以8字节对齐
    public byte n3; //1字节
    // 内存对齐 空的1字节
    public short n4; //2字节
    // 内存对齐 空的4字节
}

//由于Pack指定了1, 所以最大只能以1字节对齐
[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct Foo3 //15字节
{
    public int n1; //4字节
    public long n2; // 8字节
    public byte n3; //1字节
    public short n4; //2字节
}

struct Foo4 //32字节
{
    //虽然Int128是以两个ulong组成的, 但是它特殊
    //依旧算是16字节的一个整体, 按照16字节对齐
    public Int128 n1; //16字节
    public byte n2; //1字节
    //内存对齐 空的15字节
}

//由于Pack是8, 所以最大只会以8字节对齐
[StructLayout(LayoutKind.Sequential, Pack = 8)]
struct Foo5 //24字节
{
    public Int128 n1; //16字节
    public byte n2; //1字节
    //内存对齐 空的7字节
}
```

更详细请看这里

```csharp
// 在.NET中
unsafe struct ExampleStruct2 // 占用32字节 , 它是以8字节对齐的
{
    // 结构体最大成员是8字节, 所以以8字节对齐

    // 开始
    public byte b1; // 1
    public byte b2; // 1
    // 找到了int 按照4字节对齐, 空2字节
    public int i3; // 4
    // 集齐了8字节

    // 重新开始
    public fixed byte a4[1]; // 1
    //找到了decimal, 按照8字节对齐, 空7字节, 集齐了8字节

    // 重新开始
    // 在.NET中, decimal它里面是由int,uint,ulong组成的, 所以按那个long的8字节对齐
    public decimal d5; // 16
    // 结束, 没有空余
}

// 在.NET Framework中, decimal的定义不一样
unsafe struct ExampleStruct2 // 占用28字节 , 它是以4字节对齐的
{
    // 结构体最大成员是4字节, 所以以4字节对齐

    // 开始
    public byte b1; // 1
    public byte b2; // 1
    // 找到了int 按照4字节对齐, 空2字节, 集齐了4字节
    
    // 重新开始
    public int i3; // 4
    // 集齐了4字节

    // 重新开始
    public fixed byte a4[1]; // 1
    // 找到了decimal, 按照4字节对齐, 空3字节, 集齐了4字节

    // 重新开始
    // 在.NET Framework中, decimal它里面是由4个int组成的, 所以按照最大的int来对齐
    public decimal d5; // 16
    // 结束, 没有空余
}
```

### 取地址

在一个方法里, 所有局部变量,你都可以取地址  
所有可能会在堆上的字段都不能直接取地址, 需要fixed  
所有ref变量也需要fixed取地址

属性是不能取地址的,它通常返回的值, 如果它返回引用你就可以取地址(参考一下c++的左值和右值)  

```csharp
class Foo
{
    public string A;
    public readonly string B;
    public string C { get; set; }
    public ref string D => ref A; //只有get的返回ref string的属性
    public int[] Numbers;
    public List<int> NumberList;
    public string First1(string[] arr) => arr[0];
    public ref string First2(string[] arr) => ref arr[0];
    public ref readonly string First3(string[] arr) => ref arr[0];
    public static int F() => 1;
    public unsafe void SomeMethod<T>(string[] v1, in string v2, ref string v3, T t1)
    {
        ref var r1 = ref v1;
        Foo foo = new();
        
        var p1 = &v1; //可以, v1 v2 v3 t1都可以取地址 不需要fixed, 因为它们都在栈上, 地址是固定的
        var x1 = &1; //不可以
        var x2 = &string.Empty; //可以, 需要fixed
        var x3 = &r1; //可以, 需要fixed
        var x4 = &foo.A; //可以, 需要fixed
        var p2 = &A; //可以, 需要fixed
        var p3 = &B; //可以, 需要fixed
        var p4 = &C; //不可以
        var p5 = &v1[0].Length; //不可以
        var p6 = &v1[0]; //可以, 需要fixed
        var p7 = &D; //可以, 需要fixed
        
        //数组返回的是成员的引用, 而List的索引器和属性一样返回的是值
        var p8 = &Numbers[0]; //可以, 需要fixed
        var p9 = &NumberList[0]; //不可以
        
        var p10 = &First1(v1); //不可以
        var p11 = &First2(v1); //可以, 需要fixed
        var p12 = &First3(v1); //可以, 需要fixed
        
        var p13 = (delegate* managed<int>)&F; //可以, 不需要fixed
    }
}
```

### 注意事项

#### GC回收问题和fixed

在指针指向一个变量的时候, GC它不知道你指向了谁, 你指向的那个变量随时有可能会被回收  
当然一般在一个方法结束前是不会被回收的(有例外情况)
用fixed声明指针并且在它的范围内使用是安全的

```csharp
//这个例子可能也不太恰当
var p = F(); //这个指针p大概率是失效的
unsafe object* F()
{
    object obj = new();
    return &obj; //返回堆栈上的地址, 请永远不要这么做
}
```

```csharp
var arr = new string[] { "a", "b", "c" }
var p = GetFirstPointer(arr); //这个指针p可能随时都会失效
unsafe string* GetFirstPointer(string[] objs)
{
    fixed(string* p = &objs[0]) //在出了这个fixed的范围后,p指向的内容可能随时发生变化(被回收或者被gc移动)
    {
        return p;
    }
}
```

#### 异步不能使用unsafe

在使用了unsafe的区域, 是不能使用await关键字的, 基本上就不能使用异步方法了, ref也不能在异步方法里使用  
不过在C#13.0中可以在异步中使用ref, 但是仍然不能跨await语句

```csharp
public unsafe async Task F()
{
    int a = 1;
    ref int r = ref a; // 异步方法不能使用ref,  error: 异步方法不能有按引用局部变量
    await Task.Delay(1000); // 异步方法不能在unsafe里,  error: 无法在不安全的上下文中等待
}
```

## 函数指针

在.NET5/C#9.0 可以使用托管指针, 它不同于委托类型 `delegate*<void>`  
函数指针不能指向非静态方法, 它自己也不能拥有泛型参数, 不过它可以指向非托管代码  
函数指针的定义如下

```csharp
delegate* managed<void> f; // 无返回值和参数的托管函数指针 managed可以省略, 默认是managed
delegate* unmanaged<int> f2; //返回值类型为int的非托管函数指针
delegate* unmanaged[Stdcall]<int*,int[],ref int> //调用约定为Stdcall,参数列表为(int*,int[]) 返回值为ref int的非托管函数指针
```

获取一个函数/方法的指针的时候需要显式的指定类型, 不能使用var  
当获取一个托管方法的指针的时候, 实际上是获取的Delegate._methodPtrAux字段的值

```csharp
static unsafe int F()
{
    var fp1 = (delegate* managed<int>)&F; //正确的
    delegate* managed<int> fp2 = &F; //正确的
    var fp3 = &F; //错误的
    
    //fp1和fp2相当于获取了Delegate._methodPtrAux
    nint p = (nint)typeof(Delegate).GetField("_methodPtrAux", BindingFlags.Instance | BindingFlags.NonPublic).GetValue(F);
    Console.WriteLine(p == (nint)fp1); //True
    return 0;
}
```

对委托取地址并不是函数指针

```csharp
static unsafe void F()
{
    var f = () => 1;
    var fp = &f; //这里是Func<int>*  并不是delegate* managed<int>
    // 如果强制转换为delegate* managed<int>执行会异常
}
```

以下是Main无限递归调用

```csharp
private static unsafe void Main(string[] args)
{
    var main = (delegate*<string[],void>)&Main;
    main(args);
}
```

### 执行本机代码

可以让非托管函数指针`delegate* unmanaged<void>`指向一片内存然后执行  
需要winapi来设置内存权限, 否则申请的内存不能执行 `VirtualProtect`  
当然它也可以用来执行shellcode

```csharp
internal class Program
{
    [DllImport("kernel32.dll")]
    static extern unsafe int VirtualProtect(void* lpAddress, nuint dwSize, uint flNewProtect, out uint lpflOldProtect);
    const uint PAGE_EXECUTE_READWRITE = 0x40;
    private static unsafe void Main(string[] args)
    {
        byte[] arr = // 相当于 void f(nint* a, nint b) => *a += b;
        {
            0x48, 0x01, 0x11, //add qword ptr ds:[rcx],rdx
            0xC3, //ret
        };

        fixed (byte* p = arr)
        {
            VirtualProtect(p, (nuint)arr.Length, PAGE_EXECUTE_READWRITE, out _); //设置读写执行权限
            var f = (delegate* unmanaged<ref int, int, void>)p;
            int a = 10;
            f(ref a, 20); //执行arr里的代码
            Console.WriteLine(a); //输出30
        }
    }
}
```

### 动态调用本机函数

NativeLibrary可以用来加载外部库的函数, 调用速度和DllImport几乎一样, 不用担心速度慢  

```csharp
var kernel32 = NativeLibrary.Load("kernel32.dll");
var beepPtr = NativeLibrary.GetExport(kernel32, "Beep");
var beep = (delegate* unmanaged[Stdcall]<int, int, int>)beepPtr;
beep(1000, 1000);
```

## ref 引用

ref应该相当于托管指针, 使用它们不需要开启unsafe  
指针的话没有使用好就容易发生内存泄露甚至clr崩溃, 使用ref就可以减少这些问题

它是一个关键字, 一般用来引用一个变量, 当然也可以引用内存中的数组  
ref在方法变量声明的时候必须进行赋值

```csharp
int a = 10;
int b = 20;
ref int r = ref a; //声明一个ref变量并且指向a
ref int r2; //这是错误的, 在一个方法里声明的时候必须赋值
r = 20; //此时将a赋值为20
r = ref b; //此时r指向b, 而不是将a赋值为b
r = 40; //将b赋值为40

int* p = &a;
ref int r3 = ref *p; //指向了a
```

ref引用了一个变量后, 在这个引用存在时, 它可以保证这个变量不会被回收  
而使用指针的话, 指向的这个变量就可能会被回收  
