## Java内存模型及其目标
目标：在不牺牲性能的前提下，解决
1. 缓存 → 可见性
2. 编译/CPU重排 → 有序性
手段：“按需禁用”缓存与优化，控制权交给程序员。
1. 程序员可用接口
关键字：volatile、synchronized、final
规则：6 条 Happens-Before 关系
2. 本质（开发视角）
JMM = 规范 JVM 如何提供上述接口，让程序员精确指定何时禁用缓存、禁止重排，从而保证可见 + 有序且性能可控。

## 使用volatile的困惑
```java
// 以下代码来源于【参考1】
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() {
    x = 42;
    v = true;
  }
  public void reader() {
    if (v == true) {
      // 这里x会是多少呢？
    }
  }
}
```

JDK ≤1.4 时，volatile 只保证变量 v 本身的可见性，对普通变量 x 没有任何可见性承诺；因此线程 B 看到 v==true 时，x 仍可能读到缓存里的 0。
从 JDK 5 开始，通过新的 HB 规则（volatile 写 Happens-Before 于后续 volatile 读）+ 传递性，x 的写入被隐式带过去，于是保证读到 42。

--------------------------------------------------------------------
细节复盘（对应代码路径）

1. 线程 A  
   x = 42;        // 普通写，只停留在 A 的缓存/写缓冲  
   v = true;      // volatile 写，强制刷新 v 并插入内存屏障

2. 线程 B  
   if (v == true) // volatile 读，拿到最新值 true  
   读 x;          // 在 1.4 之前没有 HB 关系，x 仍可能走缓存旧值 → 0

3. 1.5+ 的 HB 链  
   x=42 ——(程序顺序)——> v=true ——(volatile 写)——> 后续任意 volatile 读 ——(传递性)——> 读 x  
   于是 JVM 必须在 volatile 写之后把**所有**之前普通写刷到主存，读线程因而看见 42。



## Java Happens-Before 六规则（速记版）

1. 程序顺序  
   同线程里，写在前 → 对后续代码可见。

2. volatile  
   写 volatile V → 对**后续任何线程**读 V 可见。（禁用缓存？）

3. 传递性  
   A HB B，B HB C ⇒ A HB C；用来把普通写“带”过可见性边界。
   如果线程B读到了“v=true”，那么线程A设置的“x=42”对线程B是可见的。也就是说，线程B能看到 “x == 42” ，有没有一种恍然大悟的感觉？这就是1.5版本对volatile语义的增强，这个增强意义重大，1.5版本的并发工具包（java.util.concurrent）就是靠volatile语义来搞定可见性的

   <img width="594" height="570" alt="image" src="https://github.com/user-attachments/assets/8fb2194c-ff25-41f9-baf7-3e8119dfaf3b" />

    
5. 锁（管程）  
   解锁 → 对**后续加锁**可见；synchronized 块借此保证“进去一定看见前任的全部修改”。

   管程指的是什么”。管程是一种通用的同步原语，在Java中指的就是synchronized，synchronized是Java里对管程的实现。管程中的锁在Java里是隐式实现的，例如下面的代码，在进入同步块之前，会自动加锁，而在代码块执行完会自动释放锁，加锁以及释放锁都是编译器帮我们实现的。
```java
synchronized (this) { //此处自动加锁
  // x是共享变量,初始值=10
  if (this.x < 12) {
    this.x = 12; 
  }  
} //此处自动解锁
```

  所以结合规则4——管程中锁的规则，可以这样理解：假设x的初始值是10，线程A执行完代码块后x的值会变成12（执行完自动释放锁），线程B进入代码块时，能够看到线程A对x的写操作，也就是线程B能够看到x==12。这个也是符合我们直觉的，应该不难理解。

7. start()  
   主线程启动子线程前的所有写，对子线程**第一条语句**可见。
    ```
    Thread B = new Thread(()->{
      // 主线程调用B.start()之前
      // 所有对共享变量的修改，此处皆可见
      // 此例中，var==77
    });
    // 此处对共享变量var修改
    var = 77;
    // 主线程启动子线程
    B.start();
    ```

9. join()  
   子线程里的所有写，对**join() 返回之后**的代码可见。
   ```
   Thread B = new Thread(()->{
      // 此处对共享变量var修改
      var = 66;
    });
    // 例如此处对共享变量修改，
    // 则这个修改结果对线程B可见
    // 主线程启动子线程
    B.start();
    B.join()
    // 子线程所有对共享变量的修改
    // 在主线程调用B.join()之后皆可见
    // 此例中，var==66
   ```

一句话：  
HB 不是“谁先发生”，而是“前面的结果，后面一定看得见”。

10. final
    “生而不变可劲优化，但先别让 this 逃出构造函数。”
    1.5 前：编译器重排太激进，可能把 final 字段的写放到构造函数外，其他线程看见“半成品”。
    1.5 后：JMM 加约束——只要构造函数里不把 this 逸出（不要赋给外部可见变量/注册回调/启动线程），final 字段就保证构造结束前写操作全部完成，随后任意线程读到的都是正确值。
    “逸出”有点抽象，我们还是举个例子吧，在下面例子中，在构造函数里面将this赋值给了全局变量global.obj，这就是“逸出”，线程通过global.obj读取x是有可能读到0的。因此我们一定要避免“逸出”。
    ```
    // 以下代码来源于【参考1】
    final int x;
    // 错误的构造函数
    public FinalFieldExample() { 
      x = 3;
      y = 4;
      // 此处就是讲this逸出，
      global.obj = this;
    }
    ```
