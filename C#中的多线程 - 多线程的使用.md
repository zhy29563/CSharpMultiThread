## 基于事件的异步模式

基于事件的异步模式（event-based asynchronous pattern，EAP）提供了一种简单的方式，让类可以提供多线程的能力，而不需要使用者显式启动和管理线程。它也提供如下的功能：

- 协作取消模型（cooperative cancellation model）
- 工作线程完成时[安全更新 WPF 或 Windows Forms 控件](https://blog.gkarch.com/threading/part2.html#rich-client-applications-and-thread-affinity)的能力
- 转发异常至完成事件

EAP 仅仅是一个模式，所以这些功能需要开发者自己实现。Framework 中仅有少数类采用这个模式，其中最常见的是 [`BackgroundWorker`](https://blog.gkarch.com/threading/part3.html#backgroundworker)（接下来就会讲到），以及命名空间`System.Net`中的`WebClient`。这个模式本质上就是：类提供一组成员，用于在内部管理多线程，类似于下边的代码：

```c#
// 这些成员来自于 WebClient 类:

public byte[] DownloadData (Uri address);    // 同步版本
public void DownloadDataAsync (Uri address);
public void DownloadDataAsync (Uri address, object userToken);
public event DownloadDataCompletedEventHandler DownloadDataCompleted;

public void CancelAsync (object userState);  // 取消一个操作
public bool IsBusy { get; }                  // 指示是否仍在运行
```

`*Async`方法是异步执行的：换句话说，它们在另一个线程上启动操作，然后立即返回到调用方。当操作完成时，会触发`*Completed`事件。如果是在[WPF 或 Windows Forms 应用程序](https://blog.gkarch.com/threading/part2.html#rich-client-applications-and-thread-affinity)中使用，还会自动调用`Invoke`（译者注：并不是直接调用`Invoke`，而是功能相同，可以让委托在 UI 线程执行）。这个事件传递一个事件参数对象，其包含：

- 一个表示是否操作被取消的标识（使用者调用`CancelAsync`）
- 一个`Error`对象，表示被抛出的异常（如果有异常）
- `userToken`对象（如果在调用`Asnyc`方法时提供了）

这里我们展示如何使用`WebClient`的 EAP 成员来下载一个网页：

```c#
var wc = new WebClient();
wc.DownloadStringCompleted += (sender, args) =>
{
  if (args.Cancelled)
    Console.WriteLine ("Canceled");
  else if (args.Error != null)
    Console.WriteLine ("Exception: " + args.Error.Message);
  else
  {
    Console.WriteLine (args.Result.Length + " chars were downloaded");
    // 我们可以在这里更新 UI...
  }
};
wc.DownloadStringAsync (new Uri ("http://www.linqpad.net"));  // 开始
```

采用 EAP 模式的类可能会提供额外的几组异步方法，例如：

```c#
public string DownloadString (Uri address);
public void DownloadStringAsync (Uri address);
public void DownloadStringAsync (Uri address, object userToken);
public event DownloadStringCompletedEventHandler DownloadStringCompleted;
```

这几个方法都共享相同的`CancelAsync`和`IsBusy`成员。因此，它们中同一时间只有一个异步操作能够执行。

如果 EAP 的内部实现采用 APM 模式（在[C# 4.0 in a Nutshell](http://www.albahari.com/nutshell/)的第 23 章描述），就可能会节约线程。

我们会在第 5 部分来讲[`Task`](https://blog.gkarch.com/threading/part5.html#task-parallelism)如何实现类似的功能，包括异常转发、任务延续（continuations）、取消标记以及同步上下文支持。这使得实现 EAP 就没什么吸引力了，除了在一些简单的情况下，可以直接使用[`BackgroundWorker`](https://blog.gkarch.com/threading/part3.html#backgroundworker)。

## BackgroundWorker

`BackgroundWorker`是一个命名空间`System.ComponentModel`中的工具类，用于管理工作线程。它可以被认为是一个 EAP 的通用实现，提供了下列功能：

- 协作取消模型（cooperative cancellation model）
- 工作线程完成时[安全更新 WPF 或 Windows Forms 控件](https://blog.gkarch.com/threading/part2.html#rich-client-applications-and-thread-affinity)的能力
- 转发异常至完成事件
- 报告工作进度的协议
- 实现了`IComponent`接口，使它可以在 Visual Studio 的设计器中使用

`BackgroundWorker`使用[线程池](https://blog.gkarch.com/threading/part1.html#thread-pooling)，意味着绝不应该在`BackgroundWorker`的线程上调用[`Abort`](https://blog.gkarch.com/threading/part4.html#aborting-threads)。

### 使用 BackgroundWorker

下边是使用`BackgroundWorker`的最少步骤：

1. 实例化`BackgroundWorker`并且挂接`DoWork`事件。
2. 调用`RunWorkerAsync`，可以选用`object`参数。

这样就设置好了。任何传递给`RunWorkerAsync`的参数都会被转发到`DoWork`的事件处理器（event handler），这是通过事件参数的`Argument`属性实现的。下边举例说明：

```c#
class Program
{
  static BackgroundWorker _bw = new BackgroundWorker();

  static void Main()
  {
    _bw.DoWork += bw_DoWork;
    _bw.RunWorkerAsync ("Message to worker");
    Console.ReadLine();
  }

  static void bw_DoWork (object sender, DoWorkEventArgs e)
  {
    // 这里在工作线程上执行
    Console.WriteLine (e.Argument);        // 打印 "Message to worker"
    // 执行耗时的任务...
  }
}
```

`BackgroundWorker`有一个`RunWorkerCompleted`事件，在`DoWork`事件处理器结束后触发。不是必须要处理`RunWorkerCompleted`事件，但为了查询在`DoWork`中抛出的异常，通常应该这么做。还有，`RunWorkerCompleted`事件处理器中的代码可以直接更新 UI 控件，不需要显式的封送（marshaling），而`DoWork`事件处理器中的代码则不能。

添加工作进度报告功能：

1. 设置`WorkerReportsProgress`属性为`true`。
2. 在`DoWork`事件处理器中周期性地调用`ReportProgress`方法，来报告“完成百分比”的值，以及一个可选的用户状态对象。
3. 挂接`ProgressChanged`事件，查询其事件参数的 `ProgressPercentage`属性。
4. `ProgressChanged`事件处理器中的代码可以直接与 UI 控件交互，如同`ProgressChanged`一样。一般就是在这里更新进度条控件。

添加取消功能：

1. 设置`WorkerSupportsCancellation`属性为`true`。
2. 在`DoWork`事件处理器中周期性地检查`CancellationPending`属性：如果为`true`，就设置事件参数的`Cancel`属性为`true`，然后返回。（如果工作线程认为任务工作太困难，它无法继续，此时不需要`CancellationPending`是`true`，可以直接设置`Cancel`来退出。）
3. 调用`CancelAsync`来请求取消。

下面的例子实现了前面提到的所有功能：

```c#
using System;
using System.Threading;
using System.ComponentModel;

class Program
{
  static BackgroundWorker _bw;

  static void Main()
  {
    _bw = new BackgroundWorker
    {
      WorkerReportsProgress = true,
      WorkerSupportsCancellation = true
    };
    _bw.DoWork += bw_DoWork;
    _bw.ProgressChanged += bw_ProgressChanged;
    _bw.RunWorkerCompleted += bw_RunWorkerCompleted;

    _bw.RunWorkerAsync ("Hello to worker");

    Console.WriteLine ("Press Enter in the next 5 seconds to cancel");
    Console.ReadLine();
    if (_bw.IsBusy) _bw.CancelAsync();
    Console.ReadLine();
  }

  static void bw_DoWork (object sender, DoWorkEventArgs e)
  {
    for (int i = 0; i <= 100; i += 20)
    {
      if (_bw.CancellationPending) { e.Cancel = true; return; }
      _bw.ReportProgress (i);
      Thread.Sleep (1000);      // 仅仅为了演示...
    }                           // 真实环境中不要在线程池线程上使用 Sleep ！

    e.Result = 123;    // 这会传递给 RunWorkerCompleted
  }

  static void bw_RunWorkerCompleted (object sender,
                                     RunWorkerCompletedEventArgs e)
  {
    if (e.Cancelled)
      Console.WriteLine ("You canceled!");
    else if (e.Error != null)
      Console.WriteLine ("Worker exception: " + e.Error.ToString());
    else
      Console.WriteLine ("Complete: " + e.Result);      // 来自 DoWork
  }

  static void bw_ProgressChanged (object sender,
                                  ProgressChangedEventArgs e)
  {
    Console.WriteLine ("Reached " + e.ProgressPercentage + "%");
  }
}
```

输出结果：

```
Press Enter in the next 5 seconds to cancel
Reached 0%
Reached 20%
Reached 40%
Reached 60%
Reached 80%
Reached 100%
Complete: 123

Press Enter in the next 5 seconds to cancel
Reached 0%
Reached 20%
Reached 40%

You canceled!
```

### 继承 BackgroundWorker

当你仅仅需要提供一个异步执行方法时，继承`BackgroundWorker`是一种实现[EAP](https://blog.gkarch.com/threading/part3.html#event-based-asynchronous-pattern)的简单方式。

`BackgroundWorker`不是密闭类，同时提供一个虚方法`OnDoWork`，这提供了另一种使用方式。在写一个可能很耗时的方法时，你可以多写一个版本，返回一个继承自`BackgroundWorker`的类，它本就能够并发进行工作。使用者 只需要挂接`RunWorkerCompleted`事件和`ProgressChanged`事件。比如，假设我们写过一个耗时的方法叫做`GetFinancialTotals`：

```c#
public class Client
{
Dictionary <string,int> GetFinancialTotals (int foo, int bar) { /* ... */ }
  // ...
}
```

我们可以像这样进行重构：

```c#
public class Client
{
  public FinancialWorker GetFinancialTotalsBackground (int foo, int bar)
  {
    return new FinancialWorker (foo, bar);
  }
}

public class FinancialWorker : BackgroundWorker
{
  public Dictionary <string,int> Result;   // 可以添加指定类型的字段
  public readonly int Foo, Bar;

  public FinancialWorker()
  {
    WorkerReportsProgress = true;
    WorkerSupportsCancellation = true;
  }

  public FinancialWorker (int foo, int bar) : this()
  {
    this.Foo = foo; this.Bar = bar;
  }

  protected override void OnDoWork (DoWorkEventArgs e)
  {
    ReportProgress (0, "Working hard on this report...");

    // 初始化财务报表数据
    // ...

    while (/* 计算尚未完成 */)
    {
      if (CancellationPending) { e.Cancel = true; return; }
      // 执行另一计算步骤 ...
      // ...
      ReportProgress (percentCompleteCalc, "Getting there...");
    }
    ReportProgress (100, "Done!");
    e.Result = Result = /* 计算结果 */;
  }
}
```

调用`GetFinancialTotalsBackground`会获得一个`FinancialWorker`：一个用来管理后台操作的封装，它具备真实场景的可用性。它能够报告工作进度，能够被取消，对 WPF 和 Windows Forms 应用友好，也能处理好异常。

## 中断与中止

所有[阻塞](https://blog.gkarch.com/threading/part2.html#blocking)方法（例如`Sleep`、`Join`、`EndInvoke`以及 `Wait`），在解除阻塞的条件一直未满足且没有指定超时时间的情况下，会永久阻塞。有时，可能需要提前释放一个阻塞线程，比如在程序结束的时候。有两个方法可以实现：

- `Thread.Interrupt`（中断）
- `Thread.Abort`（中止）

`Abort`方法也可以结束一个非阻塞线程，比如结束一个在进行无限循环而“卡住”的线程。`Abort`有时在合适的场景中有用，而`Interrupt`几乎不用。

`Interrupt`和`Abort`可能引起很大的麻烦：它们看起来像是解决一系列问题的明显的选择，然而正因为如此，更应该搞清楚它们可能引起的问题。

（译者注：误用中断与中止可能并没有解决问题，而只是掩盖了问题，造成更诡异的问题）

### 中断

在一个阻塞线程上调用`Interrupt`会强制释放它，并抛出一个异常`ThreadInterruptedException`异常，例如：

```c#
static void Main()
{
  Thread t = new Thread (delegate()
  {
    try { Thread.Sleep (Timeout.Infinite); }
    catch (ThreadInterruptedException) { Console.Write ("Forcibly "); }
    Console.WriteLine ("Woken!");
  });
  t.Start();
  t.Interrupt();
}
```

输出结果：

```
Forcibly Woken!
```

除非`ThreadInterruptedException`没有被处理，否则中断线程不会导致线程结束。

如果在非阻塞线程上调用`Interrupt`，线程会继续执行直到下次被阻塞，这时`ThreadInterruptedException`会被抛出。这避免了进行如下这样的测试的需要：

```
if ((worker.ThreadState & ThreadState.WaitSleepJoin) > 0)
  worker.Interrupt();
```

上边的代码不是线程安全的，因为在`if`语句和`worker.Interrupt`之间可能被抢占。

随意中断一个线程是危险的，因为调用栈上的任何框架或第三方方法可能会意外地收到中断，而并不是在你指定的代码中。只要有代码使用[锁](https://blog.gkarch.com/threading/part2.html#locking)或其它同步构造上阻塞，之前调用的中断就会作用在这里。如果该方法设计的时候没有考虑中断（在`finally`中进行适当的清理），对象就可能会成为一个不可用的状态，或者资源没能够完全释放。

而且，我们无需使用`Interrupt`：如果自己写阻塞的代码，可以通过信号构造达到相同的效果，并且更加安全，或者可以使用 Framework 4.0 的[取消标记（cancellation tokens）](https://blog.gkarch.com/threading/part3.html#safe-cancellation)。如果希望对其他人写的代码“取消阻塞”，`Abort`几乎总是更加有用。

### 中止

通过`Abort`方法也可以使[阻塞的线程](https://blog.gkarch.com/threading/part2.html#blocking)被强制释放。效果和调用`Interrupt`类似，不同的是它会抛出一个`ThreadAbortException`的异常，而不是`ThreadInterruptedException`。另外，这个异常会在`catch`块结束时被重新抛出（这是试图更好的结束线程），除非`Thread.ResetAbort`在`catch`块中被调用。在这个中间状态，[线程状态（`ThreadState`）](https://blog.gkarch.com/threading/part2.html#threadstate)是`AbortRequested`。

未处理的`ThreadAbortException`是仅有的两个不会[导致应用程序关闭](https://blog.gkarch.com/threading/part1.html#exception-handling)的异常之一。（另一个是`AppDomainUnloadException`）。

`Interrupt`和`Abort`最大的不同是：在非阻塞的线程上调用时会发生什么。调用`Interrupt`会继续工作直到下次线程被阻塞，而调用`Abort`会立即在线程正在执行的地方抛出异常（非托管代码除外）。这会是一个问题，因为 .NET Framework 中的代码可能会被中止，而其不是能够安全中止的。例如，如果中止发生在`FileStream`被构造期间，很可能造成一个非托管文件句柄会一直保持打开直到应用程序域结束。这就排除了`Abort`在几乎任何并非无足轻重的环境中的使用。

关于为什么`Abort`是不安全的更多细节，见第 4 部分的[中止线程](https://blog.gkarch.com/threading/part4.html#aborting-threads)。

然而有两种情况可以安全地使用`Abort`。第一种情况是如果你希望在中止后卸载线程的应用程序域。一个好例子是：当在写单元测试框架的时候就可以这样做。另外一种情况是，在自己的线程上你可以安全地调用`Abort`（因为你明确知道执行到了哪里）。中止你自己的线程会抛出一个“无法被吞掉”的异常：异常会在每一个`catch`块结束时被重新抛出。在 ASP.NET 中，当调用`Redirect`时就是这样做的。

[LINQPad](http://www.linqpad.net/)在你取消一个正在运行的查询时会中止线程。中止后会卸载并重建查询的应用程序域，来避免可能的状态污染。

## 安全取消

如同我们在上一节看到的，大多数情况下在线程上调用`Abort`都是危险的。替代方法是：实现一个协作（cooperative ）模式，工作线程定期检查一个用于指示是否应该中止的标识（例如[`BackgroundWorker`](https://blog.gkarch.com/threading/part3.html#backgroundworker)中讲到的）。取消的时候，发起者仅仅设置这个标识，然后等待工作线程响应。`BackgroundWorker`工具类实现了基于标识的取消模式，你也可以很容易地自己实现它。

明显的缺点是：工作线程执行的方法必须显式的支持取消。尽管如此，这是为数不多的安全取消模式之一。为说明这个模式，我们先写一个类封装取消标识：

```c#
class RulyCanceler
{
  object _cancelLocker = new object();
  bool _cancelRequest;
  public bool IsCancellationRequested
  {
    get { lock (_cancelLocker) return _cancelRequest; }
  }

  public void Cancel() { lock (_cancelLocker) _cancelRequest = true; }

  public void ThrowIfCancellationRequested()
  {
    if (IsCancellationRequested) throw new OperationCanceledException();
  }
}
```

`OperationCanceledException`是一个 Framework 的类型，仅用于这个目的。当然，其它任何异常类型也都能用。

我们可以用如下的方式使用：

```c#
class Test
{
  static void Main()
  {
    var canceler = new RulyCanceler();
    new Thread (() => {
                        try { Work (canceler); }
                        catch (OperationCanceledException)
                        {
                          Console.WriteLine ("Canceled!");
                        }
                      }).Start();
    Thread.Sleep (1000);
    canceler.Cancel();               // 安全地取消工作
  }

  static void Work (RulyCanceler c)
  {
    while (true)
    {
      c.ThrowIfCancellationRequested();
      // ...
      try      { OtherMethod (c); }
    finally  { /* 任何需要的清理 */ }
    }
  }

  static void OtherMethod (RulyCanceler c)
  {
    // 做些事情...
    c.ThrowIfCancellationRequested();
  }
}
```

我们可以简化我们的例子：去掉`RulyCanceler`类，然后给`Test`类加一个静态布尔字段`_cancelRequest`。但是，这样的话意味着如果有多个线程同时调用`Work`时，设置`_cancelRequest`为`true`会取消所有线程工作。所以，我们的`RulyCanceler`是一个有用的抽象。唯一不太优雅的地方是在我们看`Work`方法的签名时，意义可能不够明确：

```
static void Work (RulyCanceler c)
```

`Work`方法是自己要在`RulyCanceler`对象上调用`Cancel`吗？这个例子中，答案是否定的，所以如果能够通过类型系统来保证就好了。Framework 4.0 提供的取消标记（cancellation tokens）正是用于这个目的。

### 取消标记

Framework 4.0 提供了两个类来对我们之前演示的协作取消模式做了形式化：`CancellationTokenSource`和`CancellationToken`。这两个类共同工作：

- `CancellationTokenSource`定义了`Cancel`方法。
- `CancellationToken`定义了`IsCancellationRequested`属性和`ThrowIfCancellationRequested`方法。

这两个类在一起相当于一个更加复杂的`RulyCanceler`类（之前的列子中）。因为这两个类是独立的，你可以隔离取消的功能和检查取消标识的功能。

要使用这两个类，首先实例化一个`CancellationTokenSource`对象：

```
var cancelSource = new CancellationTokenSource();
```

然后，传递`Token`属性给你希望支持取消的方法：

```
new Thread (() => Work (cancelSource.Token)).Start();
```

这里是`Work`的定义：

```
void Work (CancellationToken cancelToken)
{
  cancelToken.ThrowIfCancellationRequested();
  // ...
}
```

当需要取消时，在`cancelSource`上调用`Cancel`就可以了。

`CancellationToken`是一个结构体，但是你可以把它当作类来看待。当它进行隐式复制时，副本的行为是相同的，都会引用原始的`CancellationTokenSource`。

`CancellationToken`结构体提供了其它两个有用的成员。第一个是`WaitHandle`，返回一个[等待句柄](https://blog.gkarch.com/threading/part2.html#signaling-with-event-wait-handles)，在取消时会对它发信号。第二个是`Register`，使你可以注册一个在取消时调用的委托。

取消标记在 .NET Framework 自身中也有使用，特别是在以下类中：

- [`ManualResetEventSlim`](https://blog.gkarch.com/threading/part2.html#manualresetevent)和[`SemaphoreSlim`](https://blog.gkarch.com/threading/part2.html#semaphore)
- [`CountdownEvent`](https://blog.gkarch.com/threading/part2.html#countdownevent)
- [`Barrier`](https://blog.gkarch.com/threading/part4.html#the-barrier-class)
- `BlockingCollection`
- [PLINQ](https://blog.gkarch.com/threading/part5.html#plinq)和[任务并行库（Task Parallel Library）](https://blog.gkarch.com/threading/part5.html#the-parallel-class)

这些类中的大多数都是在它们的`Wait`方法中使用取消标记。例如，如果你在`ManualResetEventSlim`上调用`Wait`的时候指定了一个取消标记，其它线程就可以调用`Cancel`来取消等待。这比在阻塞线程上调用[`Interrupt`](https://blog.gkarch.com/threading/part3.html#interrupt)要优雅和安全的多。

## 延迟初始化

多线程中一个常见的问题是如何使用线程安全的方式延迟初始化共享字段。如果你有一个字段，它的类型的构造开销很大时，就会产生这个需求：

```c#
class Foo
{
  public readonly Expensive Expensive = new Expensive();
  // ...
}
class Expensive {  /* 假设进行构造开销很大 */  }
```

这段代码的问题是在初始化`Foo`时要承担初始化`Expensive`的开销，无论`Expensive`字段是否真的会被访问。正确的方式是按需构造：

```c#
class Foo
{
  Expensive _expensive;
  public Expensive Expensive       // 延迟初始化 Expensive
  {
    get
    {
      if (_expensive == null) _expensive = new Expensive();
      return _expensive;
    }
  }
  // ...
}
```

问题又产生了，它是线程安全的吗？现在我们没有使用[锁](https://blog.gkarch.com/threading/part2.html#locking)，并且也没有使用[内存屏障](https://blog.gkarch.com/threading/part4.html#memory-barriers-and-volatility)来访问`_expensive`，考虑如果两个线程同时访问这个属性会发生什么。它们可能都会满足`if`的估值语句，然后创建了 **不同的** `Expensive`的实例。这就可能导致不可预知的错误，因此可以说通常情况下上述代码不是线程安全的。

这个问题的解决方案是在检查和初始化对象的时候使用锁：

```c#
Expensive _expensive;
readonly object _expenseLock = new object();

public Expensive Expensive
{
  get
  {
    lock (_expenseLock)
    {
      if (_expensive == null) _expensive = new Expensive();
      return _expensive;
    }
  }
}
```

### Lazy<T>

Framework 4.0 提供了一个新的类叫做`Lazy<T>`来帮助进行延迟初始化。如果使用参数`true`进行实例化，就实现了我们刚才描述的线程安全的初始化模式。

`Lazy<T>`实际上实现了一个稍高效的这个模式的版本，被称为双重检查锁（double-checked locking，双检锁）。双检锁会进行一次额外的[易失读（volatile read）](https://blog.gkarch.com/threading/part4.html#the-volatile-keyword)，在对象已经完成初始化时，能够避免获取[锁](https://blog.gkarch.com/threading/part2.html#locking)产生的开销。

使用`Lazy<T>`时，通过一个工厂方法委托来告知如何初始化新值，还有参数`true`来创建它。然后通过`Value`属性访问它的值：

```c#
Lazy<Expensive> _expensive = new Lazy<Expensive>
  (() => new Expensive(), true);

public Expensive Expensive { get { return _expensive.Value; } }
```

如果`Lazy<T>`构造器的第二个参数为`false`，它实现的是非线程安全的延迟初始化模式，就是我们在本节开始时描述的那种模式，适用于在单线程环境下使用`Lazy<T>`。

### LazyInitializer

`LazyInitializer`是一个静态类，工作方式很像`Lazy<T>`，除以下情况外：

- 它的功能是通过一组静态方法暴露的，可以直接操作你自己类型的字段。这避免了一层间接，在需要极端优化的情况下可以改善性能。
- 它提供了另一种初始化模式，来应对多个线程竞争初始化的情况。

使用`LazyInitializer`，需要在访问字段前调用`EnsureInitialized`，传递一个字段的引用和一个工厂方法委托：

```c#
Expensive _expensive;
public Expensive Expensive
{
  get          // 实现双检锁
  {
    LazyInitializer.EnsureInitialized (ref _expensive,
                                      () => new Expensive());
    return _expensive;
  }
}
```

也可以传递另一个参数来请求让参与竞争的多个线程都可以进行初始化。这听起来像我们原始的非线程安全的例子，不同之处在于第一个完成初始化的线程会胜出，所以最终仅会得到一个实例。这个技术的优点在于它比双检锁更快（在多核心情况下），因为它的实现完全不使用锁。这是一个很少需要用到的极端优化，并且会带来以下代价：

- 当参与初始化的线程数大于核心数时，它会更慢。
- 可能会因为进行了多余的初始化而浪费 CPU 资源。
- 初始化逻辑必须是线程安全的（例如，如果`Expensive`的构造器写了静态字段，就不是线程安全的）。
- 如果初始化的对象是需要进行销毁的，多余的对象需要额外的逻辑才能被销毁。

下边是双检锁的实现，以供参考：

```
volatile Expensive _expensive;
public Expensive Expensive
{
  get
  {
    if (_expensive == null)             // 第一次检查（在锁外部）
      lock (_expenseLock)
        if (_expensive == null)         // 第二次检查（在锁内部）
          _expensive = new Expensive();
    return _expensive;
  }
}
```

下边是竞争初始化（race-to-initialize）模式的实现：

```
volatile Expensive _expensive;
public Expensive Expensive
{
  get
  {
    if (_expensive == null)
    {
      var instance = new Expensive();
      Interlocked.CompareExchange (ref _expensive, instance, null);
    }
    return _expensive;
  }
}
```

## 线程局部存储

这个系列的文章大部分集中在同步构造和由线程并发访问相同数据引发的问题上。然而有时候，会希望保持数据的隔离性，确保每个线程拥有独立的副本。局部变量就可以实现这个目的，但是它们仅适用于瞬态数据。

解决方案是： 线程局部存储（thread-local storage，TLS）。你可能不得不去考虑这个需求：希望对于线程保持隔离的数据天然倾向于瞬态。它的主要应用就是储存“带外（out-of-band）”数据，来支持执行路径的基础设施，例如消息、事务以及安全令牌。在方法参数中传递这些数据是非常笨拙的，并且除了你自己的方法，其它代码无法使用。而如果用普通的静态字段来存储就意味着会在所有线程中共享它。

（译者注：原文这里写的比较抽象，实际就是说有些数据不适合作为全局的，也不适合作为方法的局部变量，这些数据是和执行路径紧密相关的。例如 ASP.NET 应用中的当前用户。不同的请求由不同的线程处理，当前用户的数据显然不能是全局的，如果通过方法传递，那可能相当多的方法都需要增加参数，所以它是和执行路径紧密相关的。理想的方案就是它对于所在执行路径是全局的，这样，在处理不同的请求时，当前用户是个即可以根据请求隔离，又类似静态的数据。（实际情况更为复杂，因为处理一个请求可能会使用多个线程，执行路径并不完全等同于线程，这里使用简化的模型来说明这种“线程局部”的需求））

线程局部存储也可以用来[优化并行代码](https://blog.gkarch.com/threading/part5.html#threadlocalt)。它让每个线程独占地访问自己的非线程安全的对象版本，这样就不需要锁，也不需要在方法调用时重新构造对象。

有三种方式实现线程局部存储。

### [ThreadStatic]

实现线程局部存储最简单的方法是使用一个静态字段，并添加`ThreadStatic`特性：

```
[ThreadStatic] static int _x;
```

这样每个线程都会拥有一个`_x`的独立副本。

不幸的是，`[ThreadStatic]`不能在实例字段上使用（添加了也无效），也和字段的初始化器配合的不好：它仅在运行静态构造方法的线程上执行一次。如果你需要使用实例字段，或者需要非默认的初始值，`ThreadLocal<T>`提供了一个更好的选择。

### ThreadLocal<T>

`ThreadLocal<T>`是 Framework 4.0 加入的。它提供了可用于静态字段和实例字段的线程局部存储，并且允许设置默认值。

下边例子描述了如何为每个线程创建一个`ThreadLocal<int>`字段，并且设置一个默认值’3’：

```
static ThreadLocal<int> _x = new ThreadLocal<int> (() => 3);
```

之后可以通过`_x`的`Value`属性获取或设置它的线程局部值。使用`ThreadLocal`的一个额外的好处是它的值是延迟初始化的：在每个线程上第一次被使用时才通过工厂方法进行初始化。

#### ThreadLocal<T> 和实例字段

`ThreadLocal<T>`也适用于实例字段和被捕获的局部变量。例如，考虑一下在多线程环境下生成随机数的问题。`Random`类不是线程安全的，所以我们要不然在使用`Random`时加锁（这样限制了并发），要不然为每个线程使用独立的`Random`对象。`ThreadLocal<T>`可以让后者的实现更简单：

```
var localRandom = new ThreadLocal<Random>(() => new Random());
Console.WriteLine (localRandom.Value.Next());
```

我们创建`Random`对象的工厂方法有点简单，使用的`Random`的无参构造方法依赖系统时间作为生成随机数的种子。在大概 10ms 时间内创建的两个`Random`对象可能会使用相同的种子，下边是解决这个问题的一个办法：

```
var localRandom = new ThreadLocal<Random>
 ( () => new Random (Guid.NewGuid().GetHashCode()) );
```

我们会在第 5 部分中用到它（见 “ PLINQ “ 中的 [并行拼写检查的例子](https://blog.gkarch.com/threading/part5.html#example-parallel-spellchecker)）。

### GetData 和 SetData

第三种方式是使用`Thread`类上的两个方法：`GetData`和`SetData`。它们在线程特定的“槽（slots）”（译者注：代表线程局部存储区中的一个位置）中存储数据。`Thread.GetData`从线程独立的数据存储区中读取数据，`Thread.SetData`向其中写数据。这两个方法都需要一个`LocalDataStoreSlot`的对象来指定这个槽。同一个槽可以跨线程使用，并且它们仍然是获取独立的值。下边是个例子：

```
class Test
{
  // 同一个 LocalDataStoreSlot 对象可以跨线程使用。
  LocalDataStoreSlot _secSlot = Thread.GetNamedDataSlot ("securityLevel");

  // 这个属性在每个线程上有独立的值。
  int SecurityLevel
  {
    get
    {
      object data = Thread.GetData (_secSlot);
      return data == null ? 0 : (int) data;    // null 相当于未初始化。
    }
    set { Thread.SetData (_secSlot, value); }
  }
  // ...
```

在这个例子中，我们调用`Thread.GetNamedDataSlot`，它创建了一个命名的槽，允许其在程序内共享。或者，你也可以通过使用未命名的槽来自行控制其作用域，用`Thread.AllocateDataSlot`来获取一个槽。

```
class Test
{
  LocalDataStoreSlot _secSlot = Thread.AllocateDataSlot();
  // ...
```

`Thread.FreeNamedDataSlot`会对所有线程释放指定的命名的槽，但是只有在所有对该槽的引用都出了其作用域，并且被垃圾回收后才会真正释放。这确保了只要保持对`LocalDataStoreSlot`对象的引用，就还能使用原来的槽，并不会因为`Thread.FreeNamedDataSlot`而失效。

## 定时器

如果你需要使用规律的时间间隔重复执行一些方法，最简单的方式是使用定时器（timer）。与下边的例子相比，定时器可以便捷、高效地使用内存和资源：

```
new Thread (delegate() {
                         while (enabled)
                         {
                           DoSomeAction();
                           Thread.Sleep (TimeSpan.FromHours (24));
                         }
                       }).Start();
```

这不仅仅会永久占用一个线程，而且如果没有额外的代码，`DoSomeAction`每天都会发生在更晚的时间。定时器解决了这些问题。

.NET Framework 提供了 4 种定时器。下边两个类是通用的多线程定时器：

- `System.Threading.Timer`
- `System.Timers.Timer`

另外两个是专用的单线程定时器：

- `System.Windows.Forms.Timer` （Windows Forms 的定时器）
- `System.Windows.Threading.DispatcherTimer` （WPF 的定时器）

多线程定时器更加强大、精确并且更加灵活，而单线程定时器对于一些简单的更新 Windows Forms 和 WPF 控件的任务来说是安全的，并且更加便捷。

### 多线程定时器

`System.Threading.Timer`是最简单的多线程定时器：它仅仅有一个构造方法和两个普通方法（取悦于极简主义者，还有本书作者！）。在接下来的例子中，一个定时器在 5 秒钟之后调用`Tick`方法来打印 “ tick… “，之后每秒打印一次直到用户按下回车键：

```
using System;
using System.Threading;

class Program
{
  static void Main()
  {
    // 首次间隔 5000ms，之后间隔 1000ms
    Timer tmr = new Timer (Tick, "tick...", 5000, 1000);
    Console.ReadLine();
    tmr.Dispose();         // 停止定时器并执行清理工作
  }

  static void Tick (object data)
  {
    // 这里运行在一个线程池线程上
    Console.WriteLine (data);          // 打印 "tick..."
  }
}
```

之后可以通过调用`Change`方法来改变定时器的时间间隔。如果你希望定时器只触发一次，可以指定`Timeout.Infinite`作为构造方法的最后一个参数。

.NET Framework 在`System.Timers`命名空间下提供了另一个名字相同的定时器类。它只是封装了 `System.Threading.Timer`，并在使用完全相同的底层引擎的前提下提供额外的便利。下面是增加功能的简介：

- 实现了`Component`，允许用于 Visual Studio 的设计器中。
- `Interval`属性代替了`Change`方法。
- `Elapsed`事件代替了回调委托。
- `Enabled`属性用于开始或停止定时器（默认值是`false`）。
- `Start`和`Stop`方法，避免对`Enabled`属性感到困惑。
- `AutoReset`标识来指定是否为可重复的事件（默认为`true`）。
- `SynchronizingObject`属性提供`Invoke`和`BeginInvoke`方法，用于[在 WPF 和 Windows Forms 控件上安全调用方法](https://blog.gkarch.com/threading/part2.html#rich-client-applications-and-thread-affinity)。

这有个例子：

```
using System;
using System.Timers;   // 命名空间是 Timers 而不是 Threading

class SystemTimer
{
  static void Main()
  {
    Timer tmr = new Timer();       // 无需任何参数
    tmr.Interval = 500;
    tmr.Elapsed += tmr_Elapsed;    // 使用事件代替委托
    tmr.Start();                   // 开启定时器
    Console.ReadLine();
    tmr.Stop();                    // 停止定时器
    Console.ReadLine();
    tmr.Start();                   // 重启定时器
    Console.ReadLine();
    tmr.Dispose();                 // 永久停止定时器
  }

  static void tmr_Elapsed (object sender, EventArgs e)
  {
    Console.WriteLine ("Tick");
  }
}
```

多线程定时器使用[线程池](https://blog.gkarch.com/threading/part1.html#thread-pooling)来允许少量线程服务多个定时器。这意味着，回调方法或`Elapsed`事件每次可能会在不同的线程上触发。此外，不论之前的`Elapsed`是否完成执行，`Elapsed`总是几乎按时触发。因此，回调方法或事件处理器必须是线程安全的。

多线程定时器的精度依赖于操作系统，通常是在 10-20 ms 的区间。如果需要更高的精度，你可以使用本地互操作（native interop）来调用 Windows 多媒体定时器，可以让精度提升到 1 ms。它定义在 *winmm.dll* 中，首先调用`timeBeginPeriod`来通知操作系统你需要更高的定时器精度，然后调用`timeSetEvent`来启动多媒体定时器。当使用完成后，调用`timeKillEvent`停止定时器，最后调用`timeEndPeriod`通知操作系统你不在需要更高的定时器精度了。可以通过搜索关键字 *dllimport winmm.dll timesetevent* 在网上找到完整的例子。

### 单线程定时器

.NET Framework 提供了两个定时器，为消除[WPF 和 Windows Forms 应用程序的线程安全问题](https://blog.gkarch.com/threading/part2.html#rich-client-applications-and-thread-affinity)而设计：

- `System.Windows.Threading.DispatcherTimer`（WPF）
- `System.Windows.Forms.Timer`（Windows Forms）

单线程定时器不是被设计成能在其特定的环境外工作的。例如，如果在 Windows 系统服务应用程序中使用 Windows Forms 定时器，`Timer`事件不会触发！

它们暴露的成员都像`System.Timers.Timer`一样（`Interval`、`Tick`、`Start`和`Stop`），并且用法也类似。但是不同之处在于其内部是如何工作的。它们不是使用[线程池](https://blog.gkarch.com/threading/part1.html#thread-pooling)来产生定时器事件，WPF 和 Windows Forms 定时器依赖于 UI 模型的底层消息循环机制（message pumping mechanism）。意味着`Tick`事件总是在创建该定时器的那个线程触发，在通常的程序中，它也就是管理所有 UI 元素和控件的那个线程。这有很多好处：

- 你可以不必考虑[线程安全](https://blog.gkarch.com/threading/part2.html#thread-safety)。
- 新的`Tick`在之前的`Tick`完成执行前不会触发。
- 你可以直接在`Tick`时间事件的处理代码中更新 UI 控件，而不需要调用`Control.Invoke`或`Dispatcher.Invoke`。

这听起来好的难以置信，直到你意识到使用这些定时器的程序并不是真正的多线程，不会有并行执行。一个线程服务于所有定时器，并且还处理 UI 事件。这带来了单线程定时器的缺点：

- 除非`Tick`事件处理器执行的很快，否则 UI 会失去响应。

这使得 WPF 和 Windows Forms 定时器仅适用于小任务，通常就是那些更新 UI 外观的任务（例如，显示时钟或倒计时）。否则，你就需要多线程定时器。

在精度方面，单线程定时器与多线程定时器类似（几十毫秒），但是通常精度更低，因为它们会被其它 UI 请求（或其它定时器事件）推迟。