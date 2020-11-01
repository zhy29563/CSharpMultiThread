## 简介及概念

C# 支持通过多线程并行执行代码，线程有其独立的执行路径，能够与其它线程同时执行。

一个 C# 客户端程序（Console 命令行、WPF 以及 Windows Forms）开始于一个单线程，这个线程（也称为“主线程”）是由 CLR 和操作系统自动创建的，并且也可以再创建其它线程。以下是一个简单的使用多线程的例子：

所有示例都假定已经引用了以下命名空间：

```c#
using System;
using System.Threading;

class ThreadTest
{
    static void Main()
    {
        Thread t = new Thread (WriteY);  // 创建新线程
        t.Start();                       // 启动新线程，执行WriteY()
        
        // 同时，在主线程做其它事情
        for (int i = 0; i < 1000; i++)
            Console.Write ("x");
    }
    
    static void WriteY()
    {
        for (int i = 0; i < 1000; i++) 
            Console.Write ("y");
    }
}
```

输出结果：

```
xxxxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxyyyyyyyyyyyyy
yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyxxxxxxxxxxxxxxxxxxxxxx
...
```

主线程创建了一个新线程`t`来不断打印字母 “ y “，与此同时，主线程在不停打印字母 “ x “。

![Starting a new Thread](C#中的多线程 - 基础知识.assets\NewThread.png)

线程一旦启动，线程的`IsAlive`属性值就会为`true`，直到线程结束。当传递给`Thread`的构造方法的委托执行完成时，线程就会结束。一旦结束，该线程不能再重新启动。

CLR 为每个线程分配各自独立的栈空间，因此局部变量是独立的。在下面的例子中，我们定义一个拥有局部变量的方法，然后在主线程和新创建的线程中同时执行该方法。

```c#
static void Main()
{
    new Thread (Go).Start();      // 在新线程执行Go()
    Go();                         // 在主线程执行Go()
}

static void Go()
{
    // 定义和使用局部变量 - 'cycles'
    for (int cycles = 0; cycles < 5; cycles++)
        Console.Write ('?');
}
```

输出结果：`??????????`

变量`cycles`的副本是分别在各自的栈中创建的，因此才会输出 10 个问号。

线程可以通过对同一对象的引用来共享数据。例如：

```
class ThreadTest
{
  bool done;

  static void Main()
  {
    ThreadTest tt = new ThreadTest();   // 创建一个公共的实例
    new Thread (tt.Go).Start();
    tt.Go();
  }

  // 注意： Go现在是一个实例方法
  void Go()
  {
     if (!done) { done = true; Console.WriteLine ("Done"); }
  }
}
```

由于两个线程是调用了同一个的`ThreadTest`实例上的`Go()`，它们共享了`done`字段，因此输出结果是一次 “ Done “，而不是两次。

输出结果：`Done`

静态字段提供了另一种在线程间共享数据的方式，以下是一个静态的`done`字段的例子：

```
class ThreadTest
{
  static bool done;    // 静态字段在所有线程中共享

  static void Main()
  {
    new Thread (Go).Start();
    Go();
  }

  static void Go()
  {
    if (!done) { done = true; Console.WriteLine ("Done"); }
  }
}
```

以上两个例子引出了一个关键概念`线程安全（thread safety`。上述两个例子的输出实际上是不确定的：” Done “ 有可能会被打印两次。如果在`Go`方法里调换指令的顺序，” Done “ 被打印两次的几率会大幅提高：

```c#
static void Go()
{
	if (!done)
    { 
    	Console.WriteLine ("Done");
        done = true; 
    }
}
```

输出结果：

```
Done
Done   (很可能!)
```



这个问题是因为一个线程对`if`中的语句估值的时候，另一个线程正在执行`WriteLine`语句，这时`done`还没有被设置为`true`。

修复这个问题需要在读写公共字段时，获得一个`排它锁（互斥锁，exclusive lock ）`。C# 提供了`lock`来达到这个目的：

```c#
class ThreadSafe
{
  static bool done;
  static readonly object locker = new object();

  static void Main()
  {
    new Thread (Go).Start();
    Go();
  }

  static void Go()
  {
    lock (locker)
    {
      if (!done) { Console.WriteLine ("Done"); done = true; }
    }
  }
}
```

当两个线程同时争夺一个锁的时候（例子中的`locker`），一个线程等待，或者说`阻塞`，直到锁变为可用。这样就确保了在同一时刻只有一个线程能进入临界区（critical section，不允许并发执行的代码），所以 “ Done “ 只被打印了一次。像这种用来避免在多线程下的不确定性的方式被称为`线程安全（thread-safe）`。

在线程间共享数据是造成多线程复杂、难以定位的错误的主要原因。尽管这通常是必须的，但应该尽可能保持简单。

一个线程被阻塞时，不会消耗 CPU 资源。

### Join 和 Sleep

可以通过调用`Join`方法来等待另一个线程结束，例如：

```c#
static void Main()
{
  Thread t = new Thread (Go);
  t.Start();
  t.Join();
  Console.WriteLine ("Thread t has ended!");
}

static void Go()
{
  for (int i = 0; i < 1000; i++) Console.Write ("y");
}
```

输出 “ y “ 1,000 次之后，紧接着会输出 “ Thread t has ended! “。当调用`Join`时可以使用一个超时参数，以毫秒或是`TimeSpan`形式。如果线程正常结束则返回`true`，如果超时则返回`false`。

`Thread.Sleep`会将当前的线程阻塞一段时间：

```c#
Thread.Sleep (TimeSpan.FromHours (1));  // 阻塞 1小时
Thread.Sleep (500);                     // 阻塞 500 毫秒
```

当使用`Sleep`或`Join`等待时，线程是`阻塞（blocked）`状态，因此不会消耗 CPU 资源。

`Thread.Sleep(0)`会立即释放当前的时间片，将 CPU 资源出让给其它线程。Framework 4.0 新的`Thread.Yield()`方法与其相同，除了它只会出让给运行在相同处理器核心上的其它线程。

`Sleep(0)`和`Yield`在调整代码性能时偶尔有用，它也是一个很好的诊断工具，可以用于找出[线程安全（thread safety）](https://blog.gkarch.com/threading/part2.html#thread-safety)的问题。如果在你代码的任意位置插入`Thread.Yield()`会影响到程序，基本可以确定存在 bug。

### 线程是如何工作的

线程在内部由一个线程调度器（thread scheduler）管理，一般 CLR 会把这个任务交给操作系统完成。线程调度器确保所有活动的线程能够分配到适当的执行时间，并且保证那些处于等待或阻塞状态（例如，等待排它锁或者用户输入）的线程不消耗CPU时间。

在单核计算机上，线程调度器会进行时间切片（time-slicing），快速的在活动线程中切换执行。在 Windows 操作系统上，一个时间片通常在十几毫秒（译者注：默认 15.625ms），远大于 CPU 在线程间进行上下文切换的开销（通常在几微秒区间）。

在多核计算机上，多线程的实现是混合了时间切片和真实的并发，不同的线程同时运行在不同的 CPU 核心上。几乎可以肯定仍然会使用到时间切片，因为操作系统除了要调度其它的应用，还需要调度自身的线程。

线程的执行由于外部因素（比如时间切片）被中断称为被抢占（preempted）。在大多数情况下，线程无法控制其在何时及在什么代码处被抢占。

### 线程 vs 进程

好比多个进程并行在计算机上执行，多个线程是在一个进程中并行执行。进程是完全隔离的，而线程是在一定程度上隔离。一般的，线程与运行在相同程序中的其它线程共享堆内存。这就是线程为何有用的部分原因，一个线程可以在后台获取数据，而另一个线程可以同时显示已获取到的数据。

### 线程的使用与误用

多线程有许多用处，下面是通常的应用场景：

- 维持用户界面的响应

  使用工作线程并行运行时间消耗大的任务，这样主UI线程就仍然可以响应键盘、鼠标的事件。

- 有效利用 CPU

  多线程在一个线程等待其它计算机或硬件设备响应时非常有用。当一个线程在执行任务时被阻塞，其它线程就可以利用这个空闲出来的CPU核心。

- 并行计算

  在多核心或多处理器的计算机上，计算密集型的代码如果通过分治策略（divide-and-conquer，见[第 5 部分](https://blog.gkarch.com/threading/part5.html)）将工作量分摊到多个线程，就可以提高计算速度。

- 推测执行（speculative execution）

  在多核心的计算机上，有时可以通过推测之后需要被执行的工作，提前执行它们来提高性能。`LINQPad`就使用了这个技术来加速新查询的创建。另一种方式就是可以多线程并行运行解决相同问题的不同算法，因为预先不知道哪个算法更好，这样做就可以尽早获得结果。

- 允许同时处理请求

  在服务端，客户端请求可能同时到达，因此需要并行处理（如果你使用 ASP.NET、WCF、Web Services 或者 Remoting，.NET Framework 会自动创建线程）。这在客户端同样有用，例如处理 P2P 网络连接，或是处理来自用户的多个请求。

如果使用了 ASP.NET 和 WCF 之类的技术，可能[不会注意到多线程被使用](https://blog.gkarch.com/threading/part2.html#thread-safety-in-application-servers)，除非是访问共享数据时（比如通过静态字段共享数据）。如果没有正确的[加锁](https://blog.gkarch.com/threading/part2.html#locking)，就[可能产生线程安全问题](https://blog.gkarch.com/threading/part2.html#thread-safety-in-application-servers)。

多线程同样也会带来缺点，最大的问题是它提高了程序的复杂度。使用多个线程本身并不复杂，复杂的是线程间的交互（一般是通过共享数据）。无论线程间的交互是否有意为之，都会带来较长的开发周期，以及带来间歇的、难以重现的 bug。因此，最好保证线程间的交互尽量少，并坚持简单和已被证明的多线程交互设计。这篇文章主要就是关于如何处理这种复杂的问题，如果能够移除线程间交互，那会轻松许多。

一个好的策略是把多线程逻辑使用可重用的类封装，以便于独立的检验和测试。.NET Framework 提供了许多高层的线程构造，之后会讲到。

当频繁地调度和切换线程时（并且如果活动线程数量大于 CPU 核心数），多线程会增加资源和 CPU 的开销，线程的创建和销毁也会增加开销。多线程并不总是能提升程序的运行速度，如果使用不当，反而可能降低速度。 例如，当需要进行大量的磁盘 I/O 时，几个工作线程顺序执行可能会比 10 个线程同时执行要快。（在[使用 `Wait` 和 `Pulse` 进行同步](https://blog.gkarch.com/threading/part4.html#signaling-with-wait-and-pulse)中，将会描述如何实现 [生产者 / 消费者队列](https://blog.gkarch.com/threading/part4.html#producer-consumer-queue)，它提供了上述功能。）

## 创建和启动线程

像我们在简介中看到的那样，使用`Thread`类的构造方法来创建线程，通过传递`ThreadStart`委托来指明线程从哪里开始运行，下面是`ThreadStart`委托的定义：

```c#
public delegate void ThreadStart();
```

调用`Start`方法后，线程开始执行，直到它所执行的方法返回后，线程终止。下面这个例子使用完整的 C# 语法创建`TheadStart`委托：

```c#
class ThreadTest
{
  static void Main()
  {
    Thread t = new Thread (new ThreadStart (Go));
    t.Start();   // 在新线程运行 GO()
    Go();        // 同时在主线程运行 GO()
  }

  static void Go()
  {
    Console.WriteLine ("hello!");
  }
}
```

在这个例子中，线程`t`执行`Go()`方法，几乎同时主线程也执行`Go()`方法，结果将打印两个 hello。

线程也可以使用更简洁的语法创建，使用方法组（method group），让 C# 编译器推断`ThreadStart`委托类型：

```c#
Thread t = new Thread (Go);    // 无需显式使用 ThreadStart
```

另一个快捷的方式是使用 lambda 表达式或者匿名方法：

```c#
static void Main()
{
  Thread t = new Thread ( () => Console.WriteLine ("Hello!") );
  t.Start();
}
```

### 向线程传递数据

向一个线程的目标方法传递参数最简单的方式是使用 lambda 表达式调用目标方法，在表达式内指定参数:

```c#
static void Main()
{
  Thread t = new Thread ( () => Print ("Hello from t!") );
  t.Start();
}

static void Print (string message)
{
  Console.WriteLine (message);
}
```

使用这种方式，可以向方法传递任意数量的参数。甚至可以将整个实现封装为一个多语句的 lambda 表达式：

```c#
new Thread (() =>
{
  Console.WriteLine ("I'm running on another thread!");
  Console.WriteLine ("This is so easy!");
}).Start();
```

在 C# 2.0 中，也可以很容易的使用匿名方法来进行相同的操作：

```c#
new Thread (delegate()
{
  ...
}).Start();
```

另一个方法是向`Thread`的`Start`方法传递参数：

```c#
static void Main()
{
  Thread t = new Thread (Print);
  t.Start ("Hello from t!");
}

static void Print (object messageObj)
{
  string message = (string) messageObj;    // 需要强制类型转换
  Console.WriteLine (message);
}
```

可以这样是因为`Thread`的构造方法通过重载来接受两个委托中的任意一个：

```c#
public delegate void ThreadStart();
public delegate void ParameterizedThreadStart (object obj);
```

`ParameterizedThreadStart`的限制是它只接受一个参数。并且由于它是`object`类型，通常需要类型转换。

#### Lambda 表达式与被捕获变量

如我们所见，lambda 表达式是向线程传递数据的最强大的方法。然而必须小心，不要在启动线程之后误修改被捕获变量（captured variables）。例如，考虑下面的例子：

```c#
for (int i = 0; i < 10; i++)
  new Thread (() => Console.Write (i)).Start();
```

输出结果是不确定的！可能是这样`0223557799`。

问题在于变量`i`在整个循环中指向相同的内存地址。所以，每一个线程在调用`Console.Write`时，都在使用这个值在运行时会被改变的变量！

类似的问题在[C# 4.0 in a Nutshell](http://www.albahari.com/nutshell/)的第 8 章的 “Captured Variables” 有描述。这个问题与多线程没什么关系，而是和 C# 的捕获变量的规则有关（在`for`和`foreach`的场景下有时不是很理想）。

解决方法就是使用临时变量，如下所示：

```c#
for (int i = 0; i < 10; i++)
{
  int temp = i;
  new Thread (() => Console.Write (temp)).Start();
}
```

变量`temp`对于每一个循环迭代是局部的。所以，每一个线程会捕获一个不同的内存地址，从而不会产生问题。我们可以使用更为简单的代码来演示前面的问题：

```c#
string text = "t1";
Thread t1 = new Thread ( () => Console.WriteLine (text) );

text = "t2";
Thread t2 = new Thread ( () => Console.WriteLine (text) );

t1.Start();
t2.Start();
```

因为两个lambda表达式捕获了相同的`text`变量，`t2`会被打印两次：

```
t2
t2
```

### 线程命名

每一个线程都有一个 Name 属性，我们可以设置它以便于调试。这在 Visual Studio 中非常有用，因为线程的名字会显示在线程窗口（Threads Window）与调试位置（Debug Location）工具栏上。线程的名字只能设置一次，以后尝试修改会抛出异常。

静态的`Thread.CurrentThread`属性会返回当前执行的线程。在下面的例子中，我们设置主线程的名字：

```
class ThreadNaming
{
  static void Main()
  {
    Thread.CurrentThread.Name = "main";
    Thread worker = new Thread (Go);
    worker.Name = "worker";
    worker.Start();
    Go();
  }

  static void Go()
  {
    Console.WriteLine ("Hello from " + Thread.CurrentThread.Name);
  }
}
```

### 前台与后台线程

默认情况下，显式创建的线程都是前台线程（foreground threads）。只要有一个前台线程在运行，程序就可以保持存活，而后台线程（background threads）并不能保持程序存活。当一个程序中所有前台线程停止运行时，仍在运行的所有后台线程会被强制终止。

线程的前台/后台状态与它的优先级和执行时间的分配无关。

可以通过线程的`IsBackground`属性来查询或修改线程的前后台状态。如下面的例子：

```
class PriorityTest
{
  static void Main (string[] args)
  {
    Thread worker = new Thread ( () => Console.ReadLine() );
    if (args.Length > 0) worker.IsBackground = true;
    worker.Start();
  }
}
```

如果这个程序以无参数的形式运行，工作线程会默认为前台，并在`ReadLine`时等待用户输入回车。此时主线程退出，但是程序仍然在运行，因为有一个前台线程依然存活。

相反，如果给`Main()`传递了参数，工作线程设置为后台状态，当主线程结束时，程序几乎立即退出（终止`ReadLine`需要一咪咪时间）。

当进程以这种方式结束时，后台线程执行栈中所有`finally`块就会被避开。如果程序依赖`finally`（或是`using`）块来执行清理工作，例如释放资源或是删除临时文件，就可能会产生问题。为了避免这种问题，在退出程序时可以显式的等待这些后台线程结束。有两种方法可以实现：

- 如果是自己创建的线程，在线程上调用[`Join`](https://blog.gkarch.com/threading/part1.html#join-and-sleep)方法。
- 如果是使用[线程池线程](https://blog.gkarch.com/threading/part1.html#thread-pooling)，使用[事件等待句柄](https://blog.gkarch.com/threading/part2.html#signaling-with-event-wait-handles)。

在任一种情况下，都应指定一个超时时间，从而可以放弃由于某种原因而无法正常结束的线程。这是后备的退出策略：我们希望程序最后可以关闭，而不是让用户去开任务管理器`(╯-_-)╯╧══╧`

如果用户使用任务管理器强行终止了 .NET 进程，所有线程都会被当作后台线程一般丢弃。这是通过观察得出的结论，并不是通过文档，而且可能会因为 CLR 和操作系统的版本不同而有不同的行为。

前台线程不需要这种处理，但是必须小心避免会使线程无法结束的 bug。程序无法正常退出的一个很有可能的原因就是仍有前台线程存在。

### 线程优先级

线程的`Priority`属性决定了相对于操作系统中的其它活动线程，它可以获得多少执行时间。线程优先级的取值如下：

```c#
enum ThreadPriority { Lowest, BelowNormal, Normal, AboveNormal, Highest }
```

只有当多个线程同时活动时，线程优先级才有意义。

在提升线程优先级前请三思，这可能会导致其它线程的资源饥饿（resource starvation，译者注：指没有分配到足够的CPU时间来运行）等问题。

提升线程的优先级是无法使其能够处理实时任务的，因为它还受到程序进程优先级的影响。要进行实时任务，必须同时使用`System.Diagnostics`中的`Process`类来提升进程的优先级（记得这不是我告诉你的）：

```c#
using (Process p = Process.GetCurrentProcess())
  p.PriorityClass = ProcessPriorityClass.High;
```

`ProcessPriorityClass.High`实际上就是一个略低于最高优先级`Realtime`的级别。将一个进程的优先级设置为`Realtime`是通知操作系统，我们绝不希望该进程将 CPU 时间出让给其它进程。如果你的程序误入一个无限循环，会发现甚至是操作系统也被锁住了，就只好去按电源按钮了`o(>_<)o`　正是由于这一原因，High 通常是实时程序的最好选择。

如果实时程序拥有用户界面，提升进程的优先级会导致大量的 CPU 时间被用于屏幕更新，这会降低整台机器的速度（特别是当 UI 很复杂时）。降低主线程的优先级，并提升进程的优先级可以保证需要进行实时任务的工作线程不会被屏幕重绘所抢占。但是这依然没有解决其它程序的CPU时间饥饿的问题，因为操作系统依然为这个进程分配了大量 CPU 资源。

理想的解决方案是分离 UI 线程和实时工作线程，使用两个进程分别运行。这样就可以分别设置各自的进程优先级，彼此之间通过 Remoting 或是内存映射文件进行通信。内存映射文件十分适用于这一任务，在[C# 4.0 in a Nutshell](http://www.albahari.com/nutshell/)的第 14 和 25 章会讲到。

即使是提升了进程优先级，托管环境在处理强实时需求时仍然有限制。除了自动垃圾回收带来的延迟，操作系统也不能够完全满足实时任务的需要，即便是非托管的程序也是如此。最好的解决办法还是使用独立的硬件或者专门的实时平台。

### 异常处理

当线程开始运行后，其创建代码所在的`try / catch / finally`块与该线程不再有任何关系。考虑下面的程序：

```
public static void Main()
{
  try
  {
    new Thread (Go).Start();
  }
  catch (Exception ex)
  {
    // 永远执行不到这里
    Console.WriteLine ("Exception!");
  }
}

static void Go() { throw null; }   // 产生 NullReferenceException 异常
```

这个例子中的`try / catch`语句是无效的，而新创建的线程将会遇到一个未处理的`NullReferenceException`。当你考虑到每一个线程具有独立的执行路径时，这种行为就可以理解了。

修改方法是将异常处理移到`Go`方法中：

```
public static void Main()
{
   new Thread (Go).Start();
}

static void Go()
{
  try
  {
    // ...
    throw null;    // 异常会在下面被捕获
    // ...
  }
  catch (Exception ex)
  {
    // 一般会记录异常， 和/或通知其它线程我们遇到问题了
    // ...
  }
}
```

在生产环境的程序中，所有线程的入口方法处都应该有一个异常处理器，就如同在主线程所做的那样（一般可能是在执行栈上靠近入口的地方）。未处理的异常会使得整个程序停止运行，弹出一个难看的对话框。

在写异常处理块的时候，最好不要忽略错误。一般应该记录异常详细信息，然后可以弹出一个对话框让用户可以选择是否自动把这些信息提交到你的服务器。最后应该关闭程序，因为很可能错误已经破坏的程序的状态。然而这么做会导致用户丢失当前的工作，比如打开的文档。

WPF 和 Windows Forms 应用中的“全局”异常处理事件（`Application.DispatcherUnhandledException`和`Application.ThreadException`）只会在主UI线程有未处理的异常时触发。对于工作线程上的异常仍然需要手动处理。

`AppDomain.CurrentDomain.UnhandledException`会对所有未处理的异常触发，但是它不提供阻止程序退出的办法。

然而在某些情况下，可以不必处理工作线程上的异常，因为 .NET Framework 会为你处理。这些会在接下来的内容中讲到：

- [异步委托](https://blog.gkarch.com/threading/part1.html#asynchronous-delegates)
- [`BackgroundWorker`](https://blog.gkarch.com/threading/part3.html#backgroundworker)
- [任务并行库（TPL）](https://blog.gkarch.com/threading/part5.html#the-parallel-class)

## 线程池

当启动一个线程时，会有几百毫秒的时间花费在准备一些额外的资源上，例如一个新的私有局部变量栈这样的事情。每个线程会占用（默认情况下）1MB 内存。线程池（thread pool）可以通过共享与回收线程来减轻这些开销，允许多线程应用在很小的粒度上而没有性能损失。在多核心处理器以分治（divide-and-conquer）的风格执行计算密集代码时将会十分有用。

线程池会限制其同时运行的工作线程总数。太多的活动线程会加重操作系统的管理负担，也会降低 CPU 缓存的效果。一旦达到数量限制，任务就会进行排队，等待一个任务完成后才会启动另一个。这使得程序任意并发成为可能，例如 web 服务器。（异步方法模式（asynchronous method pattern）是进一步高效利用线程池的高级技术，我们在[C# 4.0 in a Nutshell](http://www.albahari.com/nutshell/)的 23 章来讲）。

有多种方法可以使用线程池：

- 通过[任务并行库（TPL）](https://blog.gkarch.com/threading/part5.html#the-parallel-class)（Framework 4.0 中加入）
- 调用[`ThreadPool.QueueUserWorkItem`](https://blog.gkarch.com/threading/part1.html#queueuserworkitem)
- 通过[异步委托](https://blog.gkarch.com/threading/part1.html#asynchronous-delegates)
- 通过[`BackgroundWorker`](https://blog.gkarch.com/threading/part3.html#backgroundworker)

以下构造会间接使用线程池：

- WCF、Remoting、ASP.NET 和 ASMX 网络服务应用
- [`System.Timers.Timer`](https://blog.gkarch.com/threading/part3.html#multithreaded-timers) 和 [`System.Threading.Timer`](https://blog.gkarch.com/threading/part3.html#multithreaded-timers)
- .NET Framework 中名字以 *Async* 结尾的方法，例如`WebClient`上的方法（使用[基于事件的异步模式，EAP](https://blog.gkarch.com/threading/part3.html#the-event-based-asynchronous-pattern)），和大部分`BeginXXX`方法（异步编程模型模式，APM）
- PLINQ

任务并行库（Task Parallel Library，TPL）与 PLINQ 足够强大并面向高层，即使使用线程池并不重要，也应该使用它们来辅助多线程。我们会在[第 5 部分](https://blog.gkarch.com/threading/part5.html)中进行更详细的讨论。现在，简单看一下如何使用[`Task`](https://blog.gkarch.com/threading/part5.html#task-parallelism)类作为在线程池线程上运行委托的简单方法。

在使用线程池线程时有几点需要小心：

- 无法设置线程池线程的`Name`属性，这会令调试更为困难（当然，调试时也可以在 Visual Studio 的线程窗口中给线程附加备注）。
- 线程池线程永远是[后台线程](https://blog.gkarch.com/threading/part1.html#foreground-and-background-threads)（一般不是问题）。
- [阻塞](https://blog.gkarch.com/threading/part2.html#blocking)线程池线程可能会在程序早期带来额外的延迟，除非调用了`ThreadPool.SetMinThreads`（见[优化线程池](https://blog.gkarch.com/threading/part1.html#optimizing-the-thread-pool)）。

可以改变线程池线程的[优先级](https://blog.gkarch.com/threading/part1.html#thread-priority)，当它用完后返回线程池时会被恢复到默认状态。

可以通过`Thread.CurrentThread.IsThreadPoolThread`属性来查询当前是否运行在线程池线程上。

### 通过 TPL 使用线程池

可以很容易的使用任务并行库（Task Parallel Library，TPL）中的[`Task`](https://blog.gkarch.com/threading/part5.html#task-parallelism)类来使用线程池。`Task`类在 Framework 4.0 时被加入：如果你熟悉旧式的构造，可以将非泛型的`Task`类看作[`ThreadPool.QueueUserWorkItem`](https://blog.gkarch.com/threading/part1.html#queueuserworkitem)的替代，而泛型的`Task<TResult>`看作[异步委托](https://blog.gkarch.com/threading/part1.html#asynchronous-delegates)的替代。比起旧式的构造，新式的构造会更快速，更方便，并且更灵活。

要使用非泛型的`Task`类，调用`Task.Factory.StartNew`，并传递目标方法的委托：

```
static void Main()    // Task 类在 System.Threading.Tasks 命名空间中
{
  Task.Factory.StartNew (Go);
}

static void Go()
{
  Console.WriteLine ("Hello from the thread pool!");
}
```

`Task.Factory.StartNew`返回一个`Task`对象，可以用来监视任务，例如通过调用[Wait](https://blog.gkarch.com/threading/part5.html#waiting-on-tasks)方法来等待其结束。

当调用`Task`的[`Wait`方法](https://blog.gkarch.com/threading/part5.html#waiting-on-tasks)时，所有未处理的异常会在宿主线程被重新抛出。（如果不调用`Wait`而是丢弃不管，未处理的异常会[像普通的线程那样](https://blog.gkarch.com/threading/part1.html#exception-handling)结束程序。（译者注：在 .NET 4.5 中，为了支持基于`async / await`的异步模式，`Task`中这种“未观察”的异常默认会被忽略，而不会导致程序结束。详见[PFX 团队的博客](http://blogs.msdn.com/b/pfxteam/archive/2011/09/28/task-exception-handling-in-net-4-5.aspx)））

泛型的`Task<TResult>`类是非泛型`Task`的子类。它可以使你在其完成执行后得到一个返回值。在下面的例子中，我们使用`Task<TResult>`来下载一个网页：

```
static void Main()
{
  // 启动 task：
  Task<string> task = Task.Factory.StartNew<string>
    ( () => DownloadString ("http://www.gkarch.com") );

  // 执行其它工作，它会和 task 并行执行：
  RunSomeOtherMethod();

  // 通过 Result 属性获取返回值：
  // 如果仍在执行中, 当前进程会阻塞等待直到 task 结束：
  string result = task.Result;
}

static string DownloadString (string uri)
{
  using (var wc = new System.Net.WebClient())
    return wc.DownloadString (uri);
}
```

（这里的`<string>` 类型参数是为了示例的清晰，它可以被省略，让编译器推断。）

查询task的`Result`属性时，未处理的异常会被封装在[`AggregateException`](https://blog.gkarch.com/threading/part5.html#working-with-aggregateexception)中自动重新抛出。然而，如果没有查询`Result`属性（并且也没有调用`Wait`），未处理的异常会令程序结束。

TPL 具有更多的特性，非常适合于利用多核处理器。关于 TPL 的讨论我们在[第 5 部分](https://blog.gkarch.com/threading/part5.html#task-parallelism)中继续。

### 不通过 TPL 使用线程池

如果是使用 .NET Framework 4.0 以前的版本，则不能使用任务并行库。你必须通过一种旧的构造使用线程池：`ThreadPool.QueueUserWorkItem`与异步委托。这两者之间的不同在于异步委托可以让你从线程中返回数据，同时异步委托还可以将异常封送回调用方。

#### QueueUserWorkItem

要使用`QueueUserWorkItem`，仅需要使用希望在线程池线程上运行的委托来调用该方法：

```
static void Main()
{
  ThreadPool.QueueUserWorkItem (Go);
  ThreadPool.QueueUserWorkItem (Go, 123);
  Console.ReadLine();
}

static void Go (object data)   // 第一次调用时 data 为 null
{
  Console.WriteLine ("Hello from the thread pool! " + data);
}
```

输出结果：

```
Hello from the thread pool!
Hello from the thread pool! 123
```

目标方法`Go`，必须接受单一一个`object`参数（来满足`WaitCallback`委托）。这提供了一种向方法传递数据的便捷方式，就像`ParameterizedThreadStart`一样。与`Task`不同，`QueueUserWorkItem`并不会返回一个对象来帮助我们在后续管理其执行。并且，你必须在目标代码中显式处理异常，未处理的异常会[令程序结束](https://blog.gkarch.com/threading/part1.html#exception-handling)。

#### 异步委托

`ThreadPool.QueueUserWorkItem`并没有提供在线程执行结束之后从线程中返回值的简单机制。异步委托调用（asynchronous delegate invocations ）解决了这一问题，可以允许双向传递任意数量的参数。而且，异步委托上的未处理异常可以方便的原线程上重新抛出（更确切的说，在调用`EndInvoke`的线程上），所以它们不需要显式处理。

不要混淆异步委托和异步方法（asynchronous methods ，以 *Begin* 或 *End* 开始的方法，比如`File.BeginRead`/`File.EndRead`）。异步方法表面上看使用了相似的方式，但是其实是为了解决更困难的问题。我们在[C# 4.0 in a Nutshell](http://www.albahari.com/nutshell/)的第 23 章中描述。

下面是如何通过异步委托启动一个工作线程：

1. 创建目标方法的委托（通常是一个`Func`类型的委托）。

2. 在该委托上调用`BeginInvoke`，保存其`IAsyncResult`类型的返回值。

   `BeginInvokde`会立即返回。当线程池线程正在工作时，你可以执行其它的动作。

3. 当需要结果时，在委托上调用`EndInvoke`，传递所保存的`IAsyncResult`对象。

接下来的例子中，我们使用异步委托调用来和主线程中并行运行一个返回字行串长度的简单方法：

```c#
static void Main()
{
  Func<string, int> method = Work;
  IAsyncResult cookie = method.BeginInvoke ("test", null, null);
  //
  // 这里可以并行执行其它任务
  //
  int result = method.EndInvoke (cookie);
  Console.WriteLine ("String length is: " + result);
}

static int Work (string s) { return s.Length; }
```

`EndInvoke`会做三件事：

1. 如果异步委托还没有结束，它会等待异步委托完成执行。
2. 它会接收返回值（也包括`ref`和`out`方式的参数）。
3. 它会向调用线程抛出未处理的异常。

如果使用异步委托调用的方法没有返回值，技术上你仍然需要调用`EndInvoke`。在实践中，这里存在争论，因为不调用`EndInvoke`也不会有什么损失。然而如果你选择不调用它，就需要考虑目标方法中的异常处理来避免错误无法察觉。

（译者注：[MSDN文档](https://msdn.microsoft.com/zh-cn/library/2e08f6yc.aspx)中明确写了 “无论您使用何种方法，都要调用 **EndInvoke** 来完成异步调用。”，所以最好不要偷懒。）

当调用`BeginInvoke`时也可以指定一个回调委托。这是一个在完成时会被自动调用的、接受`IAsyncResult`对象的方法。这样可以在后面的代码中“忘记”异步委托，但是需要在回调方法里做点其它工作：

```c#
static void Main()
{
  Func<string, int> method = Work;
  method.BeginInvoke ("test", Done, method);
  // ...
  //
}

static int Work (string s) { return s.Length; }

static void Done (IAsyncResult cookie)
{
  var target = (Func<string, int>) cookie.AsyncState;
  int result = target.EndInvoke (cookie);
  Console.WriteLine ("String length is: " + result);
}
```

`BeginInvoke`的最后一个参数是一个用户状态对象，用于设置`IAsyncResult`的`AsyncState`属性。它可以是需要的任何东西，在这个例子中，我们用它向回调方法传递`method`委托，这样才能够在它上面调用`EndInvoke`。

### 优化线程池

线程池初始时其池内只有一个线程。随着任务的分配，线程池管理器就会向池内“注入”新线程来满足工作负荷的需要，直到最大数量的限制。在足够的非活动时间之后，线程池管理器在认为“回收”一些线程能够带来更好的吞吐量时进行线程回收。

可以通过调用`ThreadPool.SetMaxThreads`方法来设置线程池可以创建的线程上限；默认如下：

- Framework 4.0，32位环境下：1023
- Framework 4.0，64位环境下：32768
- Framework 3.5：每个核心 250
- Framework 2.0：每个核心 25

（这些数字可能根据硬件和操作系统不同而有差异。）数量这么多是因为要确定[阻塞](https://blog.gkarch.com/threading/part2.html#blocking)（等待一些条件，比如远程计算机的相应）的线程的条件是否被满足。

也可以通过`ThreadPool.SetMinThreads`设置线程数量下限。下限的作用比较奇妙：它是一种高级的优化技术，用来指示线程池管理器在达到下限之前不要延迟线程的分配。当存在[阻塞](https://blog.gkarch.com/threading/part2.html#blocking)线程时，提高下限可以改善程序并发性。

默认下限数量是 CPU 核心数，也就是能充分利用 CPU 的最小数值。在服务器环境下（比如 IIS 中的 ASP.NET），下限数量一般要高的多，差不多 50 或者更高。

#### 最小线程数量是如何起作用的？

将线程池的最小线程数设置为 *x* 并不是立即创建至少 *x* 个线程，而是线程会根据需要来创建。这个数值是指示线程池管理器当需要的时候，**立即** 创建 *x* 个线程。那么问题是为什么线程池在其它情况下会延迟创建线程？

答案是为了防止短生命周期的任务导致线程数量短暂高峰，使程序的内存足迹（memory footprint）快速膨胀。为了描述这个问题，考虑在一个 4 核的计算机上运行一个客户端程序，它一次发起了 40 个任务请求。如果每个任务都是一个 10ms 的计算，假设它们平均分配在 4 个核心上，总共的开销就是 100ms 多。理想情况下，我们希望这 40 个任务运行在 *4* 个线程上：

- 如果线程数量更少，就无法充分利用 4 个核心。
- 如果线程数量更多，会浪费内存和 CPU 时间去创建不必要的线程。

线程池就是以这种方式工作的。让线程数量和 CPU 核心数量匹配，就能够既保持小的内存足迹，又不损失性能。当然这也需要线程都能被充分使用（在这个例子中满足该条件）。

但是，现在来假设任务不是进行 10ms 的计算，而是请求互联网，使用半秒钟等待响应，此时本地 CPU 是空闲状态。线程池管理器的线程经济策略（译者注：指上面说的线程数量匹配核心数）这时就不灵了，应该创建更多的线程，让所有的请求同时进行。

幸运的是，线程池管理器还有一个后备方案。如果在半秒内没有能够响应请求队列，就会再创建一个新的线程，以此类推，直到线程数量上限。

半秒的等待时间是一把双刃剑。一方面它意味着一次性的短暂任务不会使程序快速消耗不必要的40MB（或者更多）的内存。另一方面，在线程池线程被阻塞时，比如在请求数据库或者调用`WebClient.DownloadFile`，就进行不必要的等待。因为这种原因，你可以通过调用`SetMinThreads`来让线程池管理器在分配最初的 *x* 个线程时不要等待，例如：

```c#
ThreadPool.SetMinThreads (50, 50);
```

（第二个参数是表示多少个线程分配给 I/O 完成端口（I/O completion ports，IOCP），来被APM使用，这会在[C# 4.0 in a Nutshell](http://www.albahari.com/nutshell/) 的第 23 章描述。）

最小线程数量的默认值是 CPU 核心数。