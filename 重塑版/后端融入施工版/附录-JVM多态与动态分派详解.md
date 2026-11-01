# 附录：多态在 JVM 里的运行机制（从一行代码到 JIT）

> 目标：跟着一行 `a.speak()`，走完 **javac → 类加载 → 运行期分派 → JIT 优化** 的完整链路，把「多态在 JVM 里到底怎么实现」讲透。
> 全文共用**一套例子**（第 1 节的 `Demo.java`）：每一节只引用其中一段，前后不换演员，读到后面忘了回第 1 节查。
> 阅读路线：1 ~ 5 是主线；6 讲 JIT 怎么把查表优化掉；7 是边界情况；8 ~ 10 是动手实验、自测和术语。

**全文地图**

| 章节 | 回答什么问题 | 用到的演员 |
|---|---|---|
| 1 | 多态在代码里长什么样 | 全部 |
| 2 | 编译期：javac 检查了什么、为什么每次调用要「选一条指令」 | `Dog.showcase()`、`ZooKeeper.poke()` |
| 3 | 类加载期：vtable / itable 怎么构建，槽位为什么能当坐标 | `Animal`、`Dog`、`Cat`、`Greeter`、`Walker` |
| 4 | 运行期：一次调用怎么走完分派 | `a.speak()` 及其特例 |
| 5 | 分派理论：重载 vs 重写，静态多分派 / 动态单分派 | `ZooKeeper.feed()` |
| 6 | JIT：内联缓存、去虚化、去优化 | `Demo.main` 的热循环 |
| 7 | 边界：分派失败时抛什么异常 | — |
| 8 | 动手实验 | 全部 |
| 9 | 一分钟自测（10 问） | — |
| 10 | 术语表 | — |

---

## 1. 全文共用的例子

先把整套源码列出来，后面每一节只取用其中一段。**不必现在看懂每一行**，知道每样东西「是干什么用的」就够了：

```java
// ===== 接口 =====
interface Greeter {
    String greet();                                              // 抽象方法

    default String greetTwice() {                                // default 方法（第 4 节）
        return greet() + " " + greet();
    }
}

interface Walker {
    void walk();                                                 // 第二个接口（为 itable 演示准备）
}

// ===== 类 =====
class Animal {
    String kind = "Animal";                                      // 字段（第 4 节「字段不参与多态」）

    void speak() { System.out.println("..."); }

    void eat(String food, int grams) { System.out.println("吃" + food); }   // 第 2 节演示字节码

    final void sleep() { System.out.println("Zzz"); }            // final 方法（第 4 节）

    static void count() { System.out.println("动物计数"); }       // 静态方法

    Animal self() { return this; }                               // 协变返回（第 3.3 节）
}

class Dog extends Animal implements Greeter, Walker {
    String kind = "Dog";                                         // 与父类同名字段（字段隐藏）

    @Override
    void speak() { System.out.println("Woof"); }

    @Override
    public String greet() { return "Woof"; }

    @Override
    public void walk() { System.out.println("Dog walks"); }

    @Override
    Dog self() { return this; }                                  // 协变返回：返回类型更具体

    private void helper() { }                                    // private 方法

    // 专门用来演示五条调用指令的方法，main 里不调用
    void showcase() {
        super.speak();                 // → invokespecial
        helper();                      // → invokespecial
        Animal.count();                // → invokestatic
        Greeter g = () -> "hi";        // → invokedynamic
        g.greet();                     // → invokeinterface
    }
}

class Cat extends Animal {
    @Override
    void speak() { System.out.println("Meow"); }
}

// ===== 重载与 invokevirtual 的演示 =====
class ZooKeeper {
    void feed(Animal a) { System.out.println("喂动物"); }          // 重载 1
    void feed(Dog d)    { System.out.println("喂狗"); }            // 重载 2

    void poke(Animal a) { a.speak(); }                            // → invokevirtual
}

// ===== 入口 =====
public class Demo {
    public static void main(String[] args) {
        Animal a = new Dog();
        Animal b = new Cat();

        a.speak();                      // Woof
        b.speak();                      // Meow

        System.out.println(a.kind);     // Animal

        ZooKeeper keeper = new ZooKeeper();
        keeper.feed(a);                 // 喂动物
    }
}
```

跑出来的结果：

| 语句 | 变量的静态类型 | 对象的真实类型 | 输出 |
|---|---|---|---|
| `a.speak()` | Animal | Dog | Woof |
| `b.speak()` | Animal | Cat | Meow |
| `a.kind` | Animal | Dog | **Animal**（字段不参与多态，第 4 节） |
| `keeper.feed(a)` | 参数 `a` 的静态类型 Animal | Dog | **喂动物**（重载按静态类型选，第 5 节） |

前两行就是多态：**同一个类型、同一句调用，执行哪段代码由对象的真实类型决定。** 后两行是它的两个反面，先埋伏笔，第 4、5 节回收。

它解决的是「扩展」问题：`main` 里只写了 `Animal`，以后加 `Bird`、`Fish` 有多少子类都不用改。但 JVM 要为此付出一个代价：**编译期不知道会调用谁，所以必须留一个「运行时再决定」的机制。** 全文就是拆解这个机制。

---

## 2. 编译期：javac 只做两件事

javac 处理 `a.speak()` 时，只干两件事，多一件都不干。

### 2.1 第一件：按静态类型做检查

编译器看 `a` 的静态类型 `Animal`：`Animal` 上有没有声明 `speak`？有，放行。它**不知道也不允许知道**运行时会指向 `Dog` 还是 `Cat`。

用例子里的 `walk()` 验证——`walk` 只声明在 `Walker` 接口上：

```java
Animal a = new Dog();

a.walk();              // ❌ 编译报错：Animal 上没有 walk（Dog 实现了，但编译器只认左边的 Animal）
((Walker) a).walk();   // ✅ 先转成 Walker 类型，编译器就能找到 walk
```

左边类型决定「能调什么」，是编译期的门禁（主附录 §1.3.2 有详细展开）。

### 2.2 第二件：为这次调用「选一条调用指令」

**先回答这个问题：什么叫「选一条调用指令」？**

Java 源码里「调用一个方法」在字节码层不是一个笼统的动作。JVM 定义了**五条专用指令**，每一条都写明「这次调用该按什么方式找目标代码」：

| 指令 | 用于 | 怎么找目标 |
|---|---|---|
| `invokestatic` | `static` 方法 | 编译期直接绑定，不查表 |
| `invokespecial` | 构造器 `<init>`、`private` 方法、`super.xxx()` | 编译期直接绑定，不查表 |
| `invokevirtual` | 普通实例方法（类） | 运行时按对象真实类型查 vtable |
| `invokeinterface` | 接口方法 | 运行时按对象真实类型查 itable |
| `invokedynamic` | lambda、字符串拼接、记录类 | 首次运行时绑定一次调用点，之后直接调 |

javac 在编译每一个方法调用时，**根据被调方法的身份，从这五条里挑一条**，把「指令 + 方法名和签名」写进 class 文件。这个决定在编译期一次性做完，之后不再改变。

第 1 节的例子刚好把五条指令全部用上了：`Dog.showcase()` 里有四条，`ZooKeeper.poke()` 里有一条。`javap -c -p` 的输出如下（指令序列和偏移是确定的；`#7` 这类常量池编号随编译环境不同而变）：

```
  void showcase();
    Code:
       0: aload_0
       1: invokespecial #7                  // Method Animal.speak:()V
       4: aload_0
       5: invokespecial #8                  // Method Dog.helper:()V
       8: invokestatic  #9                  // Method Animal.count:()V
      11: invokedynamic #12,  0              // InvokeDynamic #0:greet:()LGreeter;
      16: astore_1
      17: aload_1
      18: invokeinterface #13,  1            // InterfaceMethod Greeter.greet:()Ljava/lang/String;
      23: pop
      24: return

  void poke(Animal);
    Code:
       0: aload_1
       1: invokevirtual #14                 // Method Animal.speak:()V
       4: return

  private static java.lang.String lambda$showcase$0();   // lambda 的方法体
    Code:
       0: ldc           #2                  // String hi
       2: areturn
```

逐行对照第 1 节的源码：`super.speak()` 是 `invokespecial`，`helper()` 是 `invokespecial`，`Animal.count()` 是 `invokestatic`，lambda 是 `invokedynamic`，`g.greet()` 是 `invokeinterface`，`a.speak()`（在 `poke` 里）是 `invokevirtual`。**源码里同样都是「调用」，字节码里是五种不同指令。**

**为什么要分这么多指令，而不是统一一条？** 因为「目标能不能在编译期确定」是本质区别：

- `static`、`private`、构造器、`super` 调用：目标唯一、不可能被替换 → **静态绑定**，编译期直接把目标钉死，运行时零开销；
- 普通实例方法、接口方法：目标取决于运行时对象的真实类型 → 编译期只能留一个「名字」，运行时再查表（第 3、4 节）。

> 分清两个问题：**「选哪条指令」是一次性的编译期决定；「这条指令最终执行谁的代码」是每一次运行时的决定。** 前者是路线的选择，后者是路线上的岔路口。

### 2.3 读字节码要先懂的两件事

**① 每行左边的数字是偏移（offset）**，即这条指令在方法的 `Code` 字节数组里从第几个字节开始，不是行号。普通 `invoke*` 指令占 3 字节（`invokeinterface` / `invokedynamic` 占 5 字节），所以 `0 → 1 → 4 → 8 → 11 → 16 …` 这样跳。

**② `invoke*` 指令不负责「搬货」，只从传送带上「取货」**

每个方法的栈帧里有两块内存，分工严格：

| 内存 | 术语 | 作用 |
|---|---|---|
| 抽屉 | 局部变量表（local variable table） | 存局部变量和参数，按 0、1、2… 编号 |
| 传送带 | 操作数栈（operand stack） | 所有指令干活的地方：取货、计算、放回结果 |

**规矩：所有指令只和传送带打交道；抽屉只有 `aload`、`istore` 这类搬运指令能碰。** `invoke*` 也一样——它需要的原料（接收者、参数）必须先被搬上传送带。它的完整工作契约只有一句话：

> 从操作数栈顶部，按顺序取走「1 个接收者 + N 个参数」，执行方法，把结果放回操作数栈。

拿例子里的 `a.eat("骨头", 100)` 看全流程：

```java
Animal a = new Dog();
a.eat("骨头", 100);
```

编译后是四条指令（`javap -c`）：

```
aload_1          // 把抽屉里的 a 搬上传送带
ldc   #x         // 把字符串常量 "骨头" 搬上传送带
bipush 100       // 把数字 100 搬上传送带
invokevirtual    // Method Animal.eat:(Ljava/lang/String;I)V
```

传送带每一步的样子：

```
初始：           [空]
aload_1 后：     [a]
ldc 后：         [a, "骨头"]
bipush 后：      [a, "骨头", 100]     ← 货备齐：接收者 a 在最底下，参数按顺序叠在上面
invokevirtual：  [空]                 ← 取走 3 个包裹，执行；eat 返回 void，不放回东西
```

**为什么 `invoke` 自己不能去抽屉里取货？** 看这条指令的真身——它在 class 文件里只有 3 个字节：

```
b6 00 07        // b6 = invokevirtual 的操作码；0007 = 常量池编号 #7
```

`#7` 指向常量池条目 `Animal.eat:(Ljava/lang/String;I)V`，只回答「调用谁」这一个问题。指令里**没有任何参数在说「接收者放在哪个抽屉、参数放在哪」**，它唯一能做的事就是从传送带顶部取货。取几个？看方法签名：`(Ljava/lang/String;I)V` 表示「一个 String 参数、一个 int 参数、返回 void」，所以取 2 个参数 + 1 个接收者，共 3 个。

这不是约定俗成，是硬检查：类加载时的字节码校验器（verifier）会静态核对每条 `invoke` 执行时传送带的形状（数量、类型），对不上直接 `VerifyError`，类都加载不了。所以「先把货搬上传送带」是 JVM 的硬规矩，javac 的工作就是替你生成这些 `aload`、`ldc`、`bipush` 指令。

**一条通用规律：接收者永远最先搬、压在栈底；参数按声明顺序依次往上叠。**

回到 `showcase()` 和 `poke()` 的 javap 输出，每条调用前面的准备指令就能逐个对上：

| 调用 | 前面的准备指令 | 它们在干什么 |
|---|---|---|
| `super.speak()` | `aload_0` | 把 0 号槽的 `this` 压栈，当作这次 `invokespecial` 的接收者 |
| `helper()` | `aload_0` | 同上，private 方法也需要接收者 |
| `Animal.count()` | （无） | 静态方法没有接收者，直接 `invokestatic` |
| `Greeter g = () -> "hi"` | （无） | `invokedynamic` 本身就在栈顶生成一个 `Greeter` 对象，`astore_1` 把它存进 1 号槽（变量 g） |
| `g.greet()` | `aload_1` + 操作数 `1` | `aload_1` 把 g 压栈当接收者；第二个操作数 `1` 表示这次调用占 1 个参数位（接收者本身也算） |
| `a.speak()`（poke 里） | `aload_1` | 把参数 `a` 压栈当接收者 |

调用之后的几条指令也有含义：`invokeinterface` 的返回值（`String`）被丢掉了，所以跟一条 `pop`；lambda 表达式编译成了一个私有静态方法 `lambda$showcase$0`（方法体在输出末尾），`invokedynamic` 只是「把这个方法包装成 Greeter 对象」的入口。

注意：非静态方法的 0 号抽屉永远是 `this`（接收者作为隐含的第 0 个参数传入），参数因此从 1 号抽屉开始编号——`poke(Animal a)` 里 `this` 是 0 号、`a` 是 1 号。这也解释了为什么反复出现 `aload_1`。

> 为什么 JVM 要设计成「栈式」而不是让指令自带参数位置？因为栈式指令格式极其规整（3 字节或 5 字节），校验和执行都简单；代价是字节码里多出一堆 `aload`/`istore`，这些都由编译器算好，人不用操心。

### 2.4 指令里写的是「名字」，不是「地址」

`invokevirtual #7` 里的 `#7` 指向常量池中的一个条目：

```
#7 = Methodref  Animal.speak:()V      // 类 Animal 里，名叫 speak，签名 ()V
```

只有**类名 + 方法名 + 签名**，没有任何地址。这就像一张「欠条」：写下「欠 Animal.speak 的执行代码一份」，具体兑现要等到类加载（第 3 节）。

---

## 3. 类加载期：把「名字」兑成「坐标」

### 3.1 解析（resolution）：符号引用 → 直接引用

第一次执行到某条 invoke 指令时（HotSpot 是惰性解析），JVM 把欠条兑现：

```
① 按「类名 + 方法名 + 签名」在类元数据里找到真正的方法（Method*）
② 算出这个方法在方法表里的位置（vtable index，或 itable 的接口分区 + 方法位）
③ 把结果写进常量池缓存（CPCache）
```

**第 ③ 步是关键**：解析只做一次，之后每次调用直接命中缓存——调用方记住的是「第 3 格」这样的坐标，而不是每次重新按名字搜索。

### 3.2 vtable 的构建规则：三条

每个类在**类加载的链接阶段**生成自己的 vtable（类方法表）。构建规则只有三条：

1. 先把父类的 vtable 整个抄一份，**槽位顺序和编号原样保留**；
2. 重写的方法**替换原槽位的入口，槽位号不变**；
3. 子类新增的虚方法，**追加在末尾**。

拿第 1 节的例子画出来（`Greeter` / `Walker` 的方法走 itable，不在这里，见 3.4）：

```
Animal 的 vtable                       Dog 的 vtable
[0] toString  → Object.toString        [0] toString  → Object.toString      ← 抄来的
[1] hashCode  → Object.hashCode        [1] hashCode  → Object.hashCode      ← 抄来的
[2] equals    → Object.equals          [2] equals    → Object.equals        ← 抄来的
[3] speak     → Animal.speak           [3] speak     → Dog.speak            ← 同槽位，换入口
[4] eat       → Animal.eat             [4] eat       → Animal.eat           ← 没重写，原样抄
[5] sleep     → Animal.sleep           [5] sleep     → Animal.sleep         ← final，抄来的，但谁也换不了
[6] self      → Animal.self            [6] self      → Dog.self（桥方法）  ← 协变返回，见 3.3
```

```
Cat 的 vtable
[0] toString  → Object.toString
[1] hashCode  → Object.hashCode
[2] equals    → Object.equals
[3] speak     → Cat.speak              ← 同一个槽位，又是另一个入口
[4] eat       → Animal.eat
[5] sleep     → Animal.sleep
[6] self      → Animal.self
```

**槽位对齐是全部秘密所在**：`speak` 在任何 Animal 家族里永远是第 3 格。调用方只要记住「第 3 格」这一个常数，就能在任意子类上取到正确的入口——不用按名字搜索，不用关心对象是哪只动物。这就是「偏移量分派（offset dispatch）」这个名字的由来。

### 3.3 「重写」怎么认定

JVM 判定重写靠**签名 = 方法名 + 描述符（参数类型 + 返回类型）**，名字和参数都一致才算同一个方法。

例子里的 `self()` 用上了唯一的放宽规则：**协变返回**。

```java
class Animal { Animal self() { return this; } }
class Dog extends Animal { @Override Dog self() { return this; } }   // 合法
```

`Dog.self()` 的描述符是 `()LDog;`，和父类的 `()LAnimal;` 不同，严格说不是同一个签名。编译器的做法是：在 `Dog` 里**自动生成一个桥方法（bridge method）** `Animal self()`，内部转调 `Dog self()`，再把桥方法填进 vtable 的第 6 格——从 JVM 角度看，重写链条完好无损。

`final` 方法（例子里的 `sleep()`）不会出现在「可被替换」的集合里：字节码虽然仍是 `invokevirtual`，但 JVM 知道它不可能被任何子类重写，可以按静态绑定处理（第 6 节的 JIT 会利用这一点）。

### 3.4 itable：接口方法表，多一层查找

接口的方法表要麻烦点。类使用单继承，vtable 的一条继承链天然对齐；但一个类可以实现多个接口，接口之间**没有共享的槽位编号**。第 1 节的 `Dog` 正好实现了两个接口（`Greeter`、`Walker`），HotSpot 的 itable 布局是「分区 + 查找」：

```
Dog 的 itable
├─ 接口区（每个实现的接口一条：接口类型 → 方法区位置）
│    Greeter → 偏移 X
│    Walker  → 偏移 Y
└─ 方法区
     X 起： greet()       → Dog.greet 的入口
           greetTwice()   → Greeter 里的默认实现（Dog 没重写，登记的就是接口那页代码）
     Y 起： walk()        → Dog.walk 的入口
```

注意 `greetTwice()`：它是 `Greeter` 的 `default` 方法，`Dog` 没有重写，但 itable 里照样给它留了位置，指向接口里的方法体——这就是 4.1 节「default 方法也能被多态调用」的底层依据。

所以 `invokeinterface` 的执行要做两步：**先按接口类型在接口区定位分区，再在分区里按方法槽位取入口**。这就是「接口调用历史上比虚方法调用慢一点」的原因。

---

## 4. 运行期：一次 `a.speak()` 的分派流水线

指令、槽位、方法表全部就位后，执行 `invokevirtual` 是一条固定流水线：

```
① 从栈帧的局部变量表里取出 a 里的引用（一张写着地址的纸条）
② 顺着纸条找到堆里的对象（Dog 实例）
③ 读对象头里的 Klass Pointer，拿到对象的真实类型（Dog 的类元数据）
④ 用解析阶段缓存的 vtable index（speak = 第 3 格），从 Dog 的 vtable 取入口
⑤ 从入口开始执行 → 打印 "Woof"
```

第 ③ 步是灵魂：**对象头里的 Klass Pointer 是运行时唯一可信的「我是谁」**。编译器写在字节码里的 `Animal.speak` 在第 ③ 步之后完全不起作用——换一只 Cat，第 ④ 步自然取到 `Cat.speak`。这就是多态的全部魔法。

> **跟着追问一句：这根 Klass Pointer 是什么时候写进去的？**
>
> 不是分派时，也不是类加载时，而是**执行 `new` 指令分配对象的那一瞬间**。三个时刻要分清：
>
> | 时刻 | 发生什么 | Dog 的类元数据 / 对象头 |
> |---|---|---|
> | 类加载（第一次用到 Dog） | `.class` 读进内存，元空间生成类元数据 | 元数据诞生，全 JVM 只有一份；对象还没有 |
> | 执行 `new Dog()` 的 `new` 指令 | 分配内存、字段清零、**写对象头** | Klass Pointer 指向 Dog 的类元数据 |
> | 执行 `a.speak()`（本节的流水线） | 第 ③ 步**只读**这根指针 | 原样读取，不修改 |
>
> 注意「指针」二字：类元数据不在对象头里，对象头只存它的地址。之后对象怎么传参、怎么强转、`super.speak()` 执行父类代码，指针都不变；`a = new Cat()` 也不是改门牌，而是造一栋新房子、贴上自己的 Klass Pointer。想亲眼看到，可以用 JOL（Java Object Layout）打印对象头，`klass` 那一行就是它。

`invokeinterface` 是同样的流水线，只是第 ④ 步多一层：先按接口类型在 itable 接口区定位分区，再取方法入口。

### 4.1 特例：看起来多态、其实是静态绑定

这些都在第 1 节的源码里，逐条对照：

| 写法（例子里的位置） | 指令 | 说明 |
|---|---|---|
| `super.speak()`（`Dog.showcase`） | `invokespecial` | 明确从父类那一层的实现开始找，不参与分派；父子 `speak` 能互相调用而不死循环，全靠它 |
| `private` 方法 `helper()` | `invokespecial` | 不参与继承，不可能被重写，直接绑死 |
| `static` 方法 `Animal.count()` | `invokestatic` | 属于类不属于对象，所以 `((Animal) null).count()` 也不报 NPE |
| 构造器 `new Dog()` | `invokespecial <init>` | 目标类在 `new` 指令里就写死了 |
| `final` 方法 `sleep()` | `invokevirtual` | 指令仍是虚调用，但 JVM 知道不可能被重写，可当静态绑定优化 |
| **字段 `a.kind`** | `getfield` | **字段不参与多态**！偏移按静态类型解析，见下 |
| `default` 方法 `greetTwice()` | `invokeinterface` | 走 itable，找到「Dog 没重写、继承自 Greeter」的默认实现 |

字段不参与多态，用例子里的 `kind` 验证（`Demo.main` 里那行 `System.out.println(a.kind)`）：

```java
Animal a = new Dog();
System.out.println(a.kind);   // 输出 Animal，不是 Dog！
```

`a.kind` 按静态类型 `Animal` 解析，`getfield Animal.kind` 指向的偏移只会读到 Animal 里那个 kind。方法能被重写所以多态，字段只是同名的两个独立存储（术语叫「字段隐藏」）。

`default` 方法为什么也能多态：`Dog` 没写 `greetTwice()`，但 `Greeter` 里有默认实现，itable 在构建时把这个默认实现填进了 `Dog` 的方法区（3.4 节）。如果要在代码里**显式**调用某个接口的 default 实现（`InterfaceName.super.method()`），JVM 内部改用 MethodHandle 完成——这也是 MyBatis-Plus 代理执行 `selectVoPage` 时那段 `bindTo(proxy)` 的来源（详见主附录 §1.3.11 和 D5）。

### 4.2 NPE 抛在什么时候

```java
Animal a = null;
a.speak();   // NullPointerException
```

抛在**执行调用那一刻**：第 ① 步取出的引用是 null，根本走不到查表。所以「静态类型检查通过」和「运行不报错」是两回事，编译器只保证前者。

---

## 5. 分派理论：重载和重写走的是两条路

第 2 节说重载在编译期就定完了，第 4 节说重写按运行时类型定。这两个决定放在一起，就是分派理论的全部内容。例子还是第 1 节的 `ZooKeeper`：

```java
class ZooKeeper {
    void feed(Animal a) { System.out.println("喂动物"); }
    void feed(Dog d)    { System.out.println("喂狗"); }
}
```

### 5.1 重载：看静态类型，编译期分派

```java
ZooKeeper keeper = new ZooKeeper();
Animal a = new Dog();

keeper.feed(a);            // 输出：喂动物
keeper.feed(new Dog());    // 输出：喂狗
keeper.feed((Dog) a);      // 输出：喂狗（强转后静态类型变成 Dog）
```

第一条：对象明明是 `Dog`，却走了 `feed(Animal)`——因为重载匹配看的是参数 `a` 的**静态类型** `Animal`（第 1 节 `Demo.main` 里的输出「喂动物」就是这一行）。javac 在编译期就把「调哪个重载」拍板了，字节码里留下对应的 `invokevirtual ZooKeeper.feed:(LAnimal;)V`——**选中的签名**从此不会变；至于最终执行哪份实现，仍是运行期分派的事（比如 `ZooKeeper` 有个子类重写了 `feed(Animal)`，就会走到子类版本）。

### 5.2 静态多分派、动态单分派

学术上的「分派（dispatch）」衡量的是**选一个方法要考虑几个维度（宗量）**：

| | 选方法看什么 | 时机 | Java 结论 |
|---|---|---|---|
| 静态分派（重载） | 接收者静态类型 **+ 所有参数的静态类型** | 编译期 | **多分派** |
| 动态分派（重写） | 只有**接收者的运行时类型** | 运行期 | **单分派** |

所以：**Java 是「静态多分派、动态单分派」的语言。** `keeper.feed(a)` 实际经历两道关：

1. 编译期：按 `keeper`（静态类型 ZooKeeper）和 `a`（静态类型 Animal）选一个重载，写进指令和符号引用；
2. 运行期：`keeper.feed` 本身没有被重写，直接执行；如果被调的是 `animal.speak()` 这类虚方法，才进入第 4 节的重写分派。

重载/重写交替出现的经典陷阱：重载方法内部再调虚方法，两次分派的依据不同，结果常常出人意料。记住第 1 步只看「字面上写的类型」，就能推对。

---

## 6. JIT：把「查表」优化成「不查表」

第 4 节的流水线是**解释器**的执行方式：每次调用都老老实实走一遍。程序跑热后（调用次数触发阈值），JIT 编译器（C1 / C2）会重新编译热点方法。优化的出发点只有一个事实：**绝大多数调用点上，对象的类型其实很稳定。**

拿第 1 节的例子做实验——把 `Demo.main` 换成热循环：

```java
public static void main(String[] args) {
    Animal a = new Dog();
    for (long i = 0; i < 50_000_000L; i++) {
        a.speak();          // 这个调用点 5000 万次都指向 Dog
    }
}
```

### 6.1 第一步：收集类型反馈

解释器和 C1 给这个调用点记账：每次执行时接收者的真实类型是什么、出现了多少次。这份「类型画像（type profile）」是后续所有投机优化的依据。默认认真记录前两个类型（`-XX:TypeProfileWidth`，默认 2），更多就归为 megamorphic。

### 6.2 第二步：按类型数量分档

| 调用点见到的类型数 | 术语 | JIT 生成的处理 |
|---|---|---|
| 1 种（本例如 `Dog`） | monomorphic | 一条类型守卫 + 内联：`if (klass == Dog) 直接执行 Dog.speak 内联后的代码` |
| 2 种（如 Dog/Cat 交替） | bimorphic（polymorphic） | 两条守卫，各自内联 |
| 超过阈值 | megamorphic | 放弃内联，退回真正的 vtable 查表 |

这就是**内联缓存（inline caching）**在 JIT 里的形态：把「查表 → 跳到入口」换成「比一下身份牌 → 直接执行」。

### 6.3 第三步：类层次分析（CHA）与去虚化

更激进的一招：JIT 扫描**当前已加载的全部类**，如果 `Animal.speak` 只有一个实现 `Dog`，「这个槽位没有竞争者」就成立——直接**去虚化（devirtualization）**，把 `invokevirtual` 当 `invokespecial` 用，连守卫都省掉，甚至一路内联到底。

### 6.4 第四步：投机失败的兜底——去优化

去虚化建立在「目前只有一个实现」的前提上。一旦真的加载了 `class Bird extends Animal`：

```
依赖失效 → 受影响的编译代码标记作废（make_not_entrant）
→ 栈上正在跑的那帧退回解释器（走 uncommon trap）
→ 重新收集类型反馈，再优化
```

**JIT 的所有优化都是投机性的、随时可撤销的。** 这保证了程序和规范行为完全一致，快只是副产品——这也是初学者容易误解的点：不是「JIT 让多态变慢/变快」，而是「JIT 让大多数实际代码里的多态不花额外代价」。

### 6.5 由此得到几条实践结论

- `final` 类、`final` 方法（例子里的 `sleep()`）不需要 CHA 就能去虚化，是对 JIT 免费的礼物；
- 内联之后，「多态调用」在机器码层面可能**完全消失**，变成顺序代码——Java 高层抽象不拖性能的根本原因；
- 真正贵的是 megamorphic 调用点：单次成本约「两三次内存访问 + 一次间接跳转」，框架深处的热点常会出现这种情况；
- 观察工具：`-XX:+PrintCompilation`（看编译和去优化事件）、`-XX:+PrintInlining`（看内联与类型守卫）、JITWatch（可视化分析）。

---

## 7. 边界：分派失败时抛什么

规律：**凡是「查表」环节找不到东西，都在调用那一刻报错。**

| 异常 | 触发场景 |
|---|---|
| `AbstractMethodError` | 槽位在（父类/接口声明了方法），但没有类填实现——典型于二进制不兼容：`Greeter` 新增了方法，`Dog` 没重新编译 |
| `IncompatibleClassChangeError` | 符号引用记为方法，运行时却成了字段（或类/接口身份变了） |
| `NullPointerException` | 第 ① 步取引用就失败，根本没到查表 |
| `ClassCastException` | 发生在 `checkcast`（强转）那一刻，与分派无关，但常和「用接口类型调子类独有方法」的写法一起出现（如 `((Walker) a).walk()` 遇上 `Cat`） |

---

## 8. 动手实验

**① 看「选指令」的结果**

对第 1 节的 `Dog` 执行 `javap -c -p Dog.class`，找到 `showcase()`：`super.speak()` 是 `invokespecial`、`helper()` 是 `invokespecial`、`Animal.count()` 是 `invokestatic`、lambda 是 `invokedynamic`、`g.greet()` 是 `invokeinterface`。再对 `ZooKeeper` 执行，`poke()` 里是 `invokevirtual`。**源码里同样的「调用」写法，字节码里是五种不同指令**——这就是 2.2 节那句话的证据。

**② 看「同一字节码、不同执行结果」**

把 `Demo.main` 里的 `new Dog()` 换成 `new Cat()`，重新编译执行。`javap -c Demo` 里 `main` 方法的字节码几乎不变（只换了 `new` 的目标类），`a.speak()` 的输出却从 Woof 变成 Meow——变化全部发生在第 4 节的第 ③ 步。

**③ 看内联与去虚化**

用第 6 节的热循环运行：

```
java -XX:+PrintInlining Demo
```

观察日志：`speak` 会被内联进 main 的循环体，并带一条「klass == Dog」的类型守卫。再把循环体改成让 `a` 在 `Dog` 和 `Cat` 之间轮换，观察守卫变成两条判断链；引入更多子类，观察它退回 `invokeinterface` 查表。

**④ 字段不参与多态的验证**

跑 `Demo.main` 里的 `System.out.println(a.kind)`，确认输出 `Animal` 而不是 `Dog`；再用 `javap -c Demo` 确认那一行编译成了 `getfield Animal.kind`。

---

## 9. 一分钟自测

1. 为什么 `a.speak()` 的字节码里写的是 `Animal.speak`，实际却执行 `Dog.speak`？→ 字节码里只是符号引用（欠条），运行期按对象头 Klass Pointer 查 vtable 取入口。
2. 「选指令」和「选执行哪段代码」分别在什么时机、依据什么？→ 前者编译期，按方法身份；后者运行期，按对象真实类型。
3. 为什么 `super.speak()` 不会被分派到子类？→ `invokespecial`，编译期就指定了从父类链开始找。
4. vtable 的三条构建规则？→ 抄父类、重写换入口不换槽位、新方法追加。
5. 为什么 vtable 能用「第 N 格」做坐标？→ 槽位在继承链上对齐，重写不改变编号。
6. itable 比 vtable 多哪一步？→ 先按接口类型定位分区，再取方法入口；default 方法也登记在这里。
7. 重载和重写分别看什么类型？→ 重载看静态类型（接收者 + 参数，多分派）；重写只看接收者运行时类型（单分派）。
8. 为什么 `a.kind` 输出的是 `Animal`？→ 字段解析按静态类型定偏移，字段只有隐藏、没有重写。
9. 内联缓存分哪三档？→ monomorphic / bimorphic（polymorphic）/ megamorphic。
10. 去虚化为什么可能失效？→ 它依赖「唯一实现」的假设；新子类加载后依赖失效，相关代码去优化回退。

---

## 10. 术语表

| 术语 | 一句话 |
|---|---|
| 符号引用 | class 文件里「按名字」记录的引用，类加载时解析成直接引用 |
| 解析（resolution） | 把符号引用变成方法元数据 + 表位置的过程 |
| CPCache | 常量池缓存，保存解析结果（如 vtable index），避免重复解析 |
| vtable | 类的方法表：槽位 → 代码入口；继承对齐、重写换入口 |
| itable | 接口方法表：接口分区 + 方法区，两步查找 |
| 偏移量分派 | 用固定槽位号从 vtable 取入口的分派方式 |
| 静态绑定 | 编译期确定目标：invokestatic / invokespecial |
| 动态分派 | 运行期按对象真实类型确定目标：invokevirtual / invokeinterface |
| 静态多分派 | 编译期按接收者和参数的静态类型选重载方法 |
| 动态单分派 | 运行期只按接收者真实类型选重写方法 |
| 字段隐藏 | 子类同名字段不覆盖父类字段，按静态类型决定读哪个 |
| 内联缓存 | 用「类型守卫 + 内联」代替查表的分派优化 |
| 类型反馈（type profile） | 每个调用点记录出现过的对象类型，JIT 据此投机优化 |
| CHA | 类层次分析：扫描已加载类层次，判断某个方法是否只有一个实现 |
| 去虚化 | 把虚调用当静态调用处理（依赖 CHA 的投机） |
| 去优化 | 投机前提失效时，把编译代码作废并退回解释器 |
| 桥方法 | 编译器为协变返回等场景生成的方法，用于填 vtable 槽位 |

---

> **一句话总记**：**javac 按方法身份选一条调用指令并留下符号引用（编译期定路线）；类加载时解析出 vtable/itable 槽位（定坐标）；运行时沿对象头的真实类型查表取入口（定岔路）；JIT 再根据类型反馈把查表投机成带守卫的内联，前提失效时去优化退回（定速度）。**
