---
title: 'C#的Async & Await筆記(2)'
date: 2019-12-17 22:36:22
urlname: csharp-async-await-note-2
tags:
  - C#
  - async&await
---

既上一篇的一些觀念，來寫一些程式來實際驗證一下C#不同類型的專案上`async`/`await`跑起來會怎麼運作，執行緒會怎麼樣調用。

<!-- more -->

## TL;DR，執行緒調用方式

上一篇[C#的Async & Await筆記](../csharp-async-await-note/)有較多的概念

這一篇的程式碼在這[csharp-lab](https://github.com/ThisWayne/csharp-lab)

1. 一遇到`await`，執行緒不會馬上跳回到caller端，執行緒還是會往呼叫的methodAsync執行，直到最底層回傳一個`Task`，才開始依序跳回caller端，但如果`Task`已經是完成的狀態，則會省去原`await`機制，用原執行緒繼續執行。
2. 一般`Task`不是已經完成的狀態下，遇到`await`，會優先看「當前」執行緒上有沒有`SynchronizationContext`，有則`await`後續的code會是原執行緒main thread來執行；沒有則`await`後續的code由thread pool裡面的執行緒執行。
3. 一般`Task`不是已經完成的狀態下，遇到`await`加上`ConfigureAwait(false)`，則`await`後續的code會由thread pool裡面的執行緒執行。

## 執行環境

### 硬體

* CPU：4 * Intel i7-6600U 2.6GHz
* RAM： 16.0 GB
* 系統類型： x64
* 作業系統：Windows 10

### 測試的project類型與target framework

* Console App： .NET Framework 4.6.1
* WPF： .NET Framework 4.6.1
* .NET Framework Web API： .NET Framework 4.6.1
* .NET Core Web API： .NET Core 2.1.1

## 測試方式

1. 寫了一個library project `TestClassLibrary`裡面放一個`AsyncAwaitTestClass.cs`，所有要跑的code都寫在裡面，可以給.NET Framework和.NET Core參照與使用
2. 下面跑測試會個別跑，會先註解掉其他測試來讓執行測試的起始情況一致
3. 測試有寫的四種project類型，測試上不一樣的地方主要是WPF的main thread有`SynchronizationContext`，其他測試上的差別只有起始thread像WPF、Console App是main thread，還是像web server是調用worker thread的差別
4. 因為有無`SynchronizationContext`，基本上只會有兩種較不一樣的執行結果，下面只寫出WPF和.NET Core的結果，想執行看看都還是可以抓回去玩玩看
5. 印出測試訊息用`Debug.WriteLine`寫在output讓不同的project類型都會在同一個地方印出來，會透過執行`AsyncAwaitTestClass.PrintInfos`印出執行的當下有多少worker threads、iocp threads、total threads，印出當下執行緒的`SynchronizationContext`、`ManagedThreadId`、`IsThreadPoolThread`。

```csharp
private void PrintInfos()
{
    ThreadPool.GetMaxThreads(out int maxWorkerThreads, out int maxCompletionPortThreads);
    ThreadPool.GetAvailableThreads(out int workerThreads, out int completionPortThreads);
    Debug.WriteLine(
        "  -- Worker threads: {0}, Completion port threads: {1}, Total threads: {2}",
        maxWorkerThreads - workerThreads,
        maxCompletionPortThreads - completionPortThreads,
        Process.GetCurrentProcess().Threads.Count
    );
    Debug.WriteLine(
        $"     SynchronizationContext: {SynchronizationContext.Current}\n" +
        $"     ManagedThreadId: {Thread.CurrentThread.ManagedThreadId}\n" +
        $"     IsThreadPoolThread: {Thread.CurrentThread.IsThreadPoolThread}"
    );
}
```

## 測試一：執行續一遇到`await`就返回呼叫端？

### (1)程式碼

```csharp
private async Task RunTest1()
{
    Debug.WriteLine("===== Test 1 =====");

    Debug.WriteLine("1. === Before ReturnFinishedTaskAsync ===");
    PrintInfos();
    await ReturnFinishedTaskAsync();

    Debug.WriteLine("===== Test 1 End =====");
}

private async Task<int> ReturnFinishedTaskAsync()
{
    Debug.WriteLine("2.   === Begin MethodWithFinishedTask ===");
    PrintInfos();
    return await Task.FromResult<int>(0);
}
```

### (1)執行結果

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 10
     SynchronizationContext: 
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test Start =====
===== Test 1 =====
1. === Before ReturnFinishedTaskAsync ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 10
     SynchronizationContext: 
     ManagedThreadId: 1
     IsThreadPoolThread: False
2.   === Begin MethodWithFinishedTask ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 10
     SynchronizationContext: 
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 1 End =====
```

不是一看到`await`，當前thread就跳回到caller，`await`裡面的code還是會先由當前thread執行，會直到執行到底層傳回task才「通常」開始一層一層返回caller。

可以在output上看到`RunTest1`的`1.`跟`ReturnFinishedTaskAsync`裡面的`2.`都是同一個thread ID，四種project結果都一樣。
其實想成下面這樣，程式是一樣的，應該就直覺async method裡面也會是同一個thread先執行。

```csharp
private async Task RunTest1()
{
    Debug.WriteLine("===== Test 1 =====");

    Debug.WriteLine("1. === Before ReturnFinishedTaskAsync ===");
    PrintInfos();
    var task = ReturnFinishedTaskAsync(); // 這裡把await分開來寫
    await task;

    Debug.WriteLine("===== Test 1 End =====");
}
```

## 測試二：`await`加不加`ConfigureAwait(false)`會發生什麼事，後續執行緒是？

### (2.1)不加`ConfigureAwait(false)`

#### (2.1)程式碼

```csharp
private async Task RunTest2_1()
{
    Debug.WriteLine("===== Test 2.1 =====");

    Debug.WriteLine("1. === Before httpClient.GetStringAsync ===");
    PrintInfos();

    await httpClient.GetStringAsync(url); // without ConfigureAwait(false)

    Debug.WriteLine("2. === After httpClient.GetStringAsync  ===");
    PrintInfos();
}
```

#### (2.1)執行結果

##### (2.1)WPF

執行`httpClient.GetStringAsync`前後的code會是一樣的thread

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 2.1 =====
1. === Before httpClient.GetStringAsync ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
2. === After httpClient.GetStringAsync  ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 28
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
```

##### (2.1).NET Core Web API

執行`httpClient.GetStringAsync`前後的code是不一樣的thread，後面是iocp thread

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 1, Completion port threads: 0, Total threads: 23
     SynchronizationContext: 
     ManagedThreadId: 16
     IsThreadPoolThread: True
===== Test 2.1 =====
1. === Before httpClient.GetStringAsync ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 23
     SynchronizationContext: 
     ManagedThreadId: 16
     IsThreadPoolThread: True
2. === After httpClient.GetStringAsync  ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 24
     SynchronizationContext: 
     ManagedThreadId: 22
     IsThreadPoolThread: True
```

### (2.2)加`ConfigureAwait(false)`

#### (2.2)程式碼

```csharp
private async Task RunTest2_2()
{
    Debug.WriteLine("===== Test 2.2 =====");

    Debug.WriteLine("1. === Before httpClient.GetStringAsync ===");
    PrintInfos();

    await httpClient.GetStringAsync(url).ConfigureAwait(false); // with ConfigureAwait(false)

    Debug.WriteLine("2. === After httpClient.GetStringAsync ConfigureAwait(false) ===");
    PrintInfos(); // 1. and 2. would be different thread ID
}
```

#### (2.2)執行結果

##### (2.2)WPF

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 14
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 2.2 =====
1. === Before httpClient.GetStringAsync ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 14
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
2. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 26
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
```

##### (2.2).NET Core Web API

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 1, Completion port threads: 0, Total threads: 28
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
===== Test 2.2 =====
1. === Before httpClient.GetStringAsync ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 28
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
2. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 30
     SynchronizationContext: 
     ManagedThreadId: 12
     IsThreadPoolThread: True
```

### (2.3)加`ConfigureAwait(false)`後面一定會是iocp thread？

#### (2.3)程式碼

```csharp
private async Task RunTest2_3()
{
    Debug.WriteLine("===== Test 2.3 =====");

    Debug.WriteLine("1. === Before ReturnFinishedTaskAsync ===");
    PrintInfos();

    await ReturnFinishedTaskAsync().ConfigureAwait(false); // with ConfigureAwait(false)

    Debug.WriteLine("3. === After ReturnFinishedTaskAsync ConfigureAwait(false) ===");
    PrintInfos();
}
```

#### (2.3)執行結果

如果跑的不是非同步IO當然不會是iocp thread

如果`Task`已經是完成的狀態，則會省去原`await`機制，用原執行緒繼續執行

##### (2.3)WPF

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 2.3 =====
1. === Before ReturnFinishedTaskAsync ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
2.   === Begin MethodWithFinishedTask ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
3. === After ReturnFinishedTaskAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
```

##### (2.3).NET Core Web API

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 1, Completion port threads: 0, Total threads: 23
     SynchronizationContext: 
     ManagedThreadId: 9
     IsThreadPoolThread: True
===== Test 2.3 =====
1. === Before ReturnFinishedTaskAsync ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 23
     SynchronizationContext: 
     ManagedThreadId: 9
     IsThreadPoolThread: True
2.   === Begin MethodWithFinishedTask ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 23
     SynchronizationContext: 
     ManagedThreadId: 9
     IsThreadPoolThread: True
3. === After ReturnFinishedTaskAsync ConfigureAwait(false) ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 23
     SynchronizationContext: 
     ManagedThreadId: 9
     IsThreadPoolThread: True
```

### (2.4)連續兩個`await`呼叫有加`ConfigureAwait(false)`

#### (2.4)程式碼

```csharp
private async Task RunTest2_4()
{
    Debug.WriteLine("===== Test 2.4 =====");

    Debug.WriteLine("1. === Before httpClient.GetStringAsync ===");
    PrintInfos();

    await httpClient.GetStringAsync(url).ConfigureAwait(false); // with ConfigureAwait(false)

    Debug.WriteLine("2. === After httpClient.GetStringAsync ConfigureAwait(false) ===");
    PrintInfos();

    await httpClient.GetStringAsync(url).ConfigureAwait(false); // with ConfigureAwait(false)

    Debug.WriteLine("3. === After httpClient.GetStringAsync ConfigureAwait(false) ===");
    PrintInfos();
}
```

#### (2.4)執行結果

這邊訊息的2.跟3.有可能會是同一個iocp thread，畢竟我都對同一個URL發HTTP GET，有可能是cache導致`Task`很快就完成了，或是真的第二個`await`真的接手的thread跟第一個一樣

##### (2.4)WPF

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 2.4 =====
1. === Before httpClient.GetStringAsync ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
2. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 28
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
3. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 28
     SynchronizationContext: 
     ManagedThreadId: 12
     IsThreadPoolThread: True
```

##### (2.4).NET Core Web API

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 1, Completion port threads: 0, Total threads: 29
     SynchronizationContext: 
     ManagedThreadId: 15
     IsThreadPoolThread: True
===== Test 2.4 =====
1. === Before httpClient.GetStringAsync ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 29
     SynchronizationContext: 
     ManagedThreadId: 15
     IsThreadPoolThread: True
2. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 30
     SynchronizationContext: 
     ManagedThreadId: 20
     IsThreadPoolThread: True
3. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 30
     SynchronizationContext: 
     ManagedThreadId: 17
     IsThreadPoolThread: True
```

### (2.5)先呼叫一個`await`有加`ConfigureAwait(false)`，然後再呼叫一個`await`不加`ConfigureAwait(false)`？

#### (2.5)程式碼

```csharp
private async Task RunTest2_5()
{
    Debug.WriteLine("===== Test 2.5 =====");

    Debug.WriteLine("1. === Before httpClient.GetStringAsync ===");
    PrintInfos();

    await httpClient.GetStringAsync(url).ConfigureAwait(false); // with ConfigureAwait(false)

    Debug.WriteLine("2. === After httpClient.GetStringAsync ConfigureAwait(false) ===");
    PrintInfos();

    await httpClient.GetStringAsync(url); // without ConfigureAwait(false)

    Debug.WriteLine("3. === After httpClient.GetStringAsync ===");
    PrintInfos();
}
```

#### (2.5)執行結果

WPF的第一個`httpClient.GetStringAsync`有加`ConfigureAwait(false)`，所以`await`接手的會是iocp thread，可以看到2.的訊息顯示沒有`SynchronizationContext`，所以第二個`httpClient.GetStringAsync`即使沒有加`ConfigureAwait(false)`，後續的code也不會是main thread

##### (2.5)WPF

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 20
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 2.5 =====
1. === Before httpClient.GetStringAsync ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 20
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
2. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 28
     SynchronizationContext: 
     ManagedThreadId: 12
     IsThreadPoolThread: True
3. === After httpClient.GetStringAsync ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 28
     SynchronizationContext: 
     ManagedThreadId: 11
     IsThreadPoolThread: True
```

##### (2.5).NET Core Web API

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 1, Completion port threads: 0, Total threads: 25
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
===== Test 2.5 =====
1. === Before httpClient.GetStringAsync ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 25
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
2. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 30
     SynchronizationContext: 
     ManagedThreadId: 12
     IsThreadPoolThread: True
3. === After httpClient.GetStringAsync ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 30
     SynchronizationContext: 
     ManagedThreadId: 12
     IsThreadPoolThread: True
```

### (2.6)外面的沒加`ConfigureAwait(false)`，裡面的有加

#### (2.6)程式碼

```csharp
private async Task RunTest2_6()
{
    Debug.WriteLine("===== Test 2.6 =====");

    Debug.WriteLine("1. === Before MethodWithConfigureAwaitFalseInsideAsync ===");
    PrintInfos();

    await MethodWithConfigureAwaitFalseInsideAsync();

    Debug.WriteLine("4. === After MethodWithConfigureAwaitFalseInsideAsync ===");
    PrintInfos();
}

private async Task<string> MethodWithConfigureAwaitFalseInsideAsync()
{
    Debug.WriteLine("2. === Before httpClient.GetStringAsync ConfigureAwait(false) ===");
    PrintInfos();

    var result = await httpClient.GetStringAsync(url).ConfigureAwait(false);

    Debug.WriteLine("3. === After httpClient.GetStringAsync ConfigureAwait(false) ===");
    PrintInfos();

    return result;
}
```

#### (2.6)執行結果

在外層WPF因為有`SynchronizationContext`所以`await`後續還是main thread，.NET Core沒有`SynchronizationContext`所以裡面最後是iocp thread，外層也是iocp thread繼續 

##### (2.6)WPF

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 2.6 =====
1. === Before MethodWithConfigureAwaitFalseInsideAsync ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
2. === Before httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
3. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 28
     SynchronizationContext: 
     ManagedThreadId: 11
     IsThreadPoolThread: True
4. === After MethodWithConfigureAwaitFalseInsideAsync ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 28
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
```

##### (2.6).NET Core Web API

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 1, Completion port threads: 0, Total threads: 29
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
===== Test 2.6 =====
1. === Before MethodWithConfigureAwaitFalseInsideAsync ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 29
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
2. === Before httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 29
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
3. === After httpClient.GetStringAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 30
     SynchronizationContext: 
     ManagedThreadId: 16
     IsThreadPoolThread: True
4. === After MethodWithConfigureAwaitFalseInsideAsync ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 30
     SynchronizationContext: 
     ManagedThreadId: 16
     IsThreadPoolThread: True
```

### (2.7)外面的有加`ConfigureAwait(false)`，裡面的沒加

#### (2.7)程式碼

```csharp
private async Task RunTest2_7()
{
    Debug.WriteLine("===== Test 2.7 =====");

    Debug.WriteLine("1. === Before MethodWithoutConfigureAwaitFalseInsideAsync ConfigureAwait(false) ===");
    PrintInfos();

    await MethodWithoutConfigureAwaitFalseInsideAsync().ConfigureAwait(false);

    Debug.WriteLine("4. === After MethodWithoutConfigureAwaitFalseInsideAsync ConfigureAwait(false) ===");
    PrintInfos();
}

private async Task<string> MethodWithoutConfigureAwaitFalseInsideAsync()
{
    Debug.WriteLine("2. === Before httpClient.GetStringAsync ===");
    PrintInfos();

    var result = await httpClient.GetStringAsync(url);

    Debug.WriteLine("3. === After httpClient.GetStringAsync ===");
    PrintInfos();

    return result;
}
```

#### (2.7)執行結果

WPF因為裡面的沒加`ConfigureAwait(false)`，所以`await`後續接手會是main thread，裡面最後執行完的是main thread，外層的有加`ConfigureAwait(false)`，不交由main thread執行`ConfigureAwait(false)`的後續，所以由worker thread接手

##### (2.7)WPF

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 2.7 =====
1. === Before MethodWithoutConfigureAwaitFalseInsideAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
2. === Before httpClient.GetStringAsync ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
3. === After httpClient.GetStringAsync ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 28
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
4. === After MethodWithoutConfigureAwaitFalseInsideAsync ConfigureAwait(false) ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 28
     SynchronizationContext: 
     ManagedThreadId: 5
     IsThreadPoolThread: True
```

##### (2.7).NET Core Web API

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 1, Completion port threads: 0, Total threads: 31
     SynchronizationContext: 
     ManagedThreadId: 9
     IsThreadPoolThread: True
===== Test 2.7 =====
1. === Before MethodWithoutConfigureAwaitFalseInsideAsync ConfigureAwait(false) ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 31
     SynchronizationContext: 
     ManagedThreadId: 9
     IsThreadPoolThread: True
2. === Before httpClient.GetStringAsync ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 31
     SynchronizationContext: 
     ManagedThreadId: 9
     IsThreadPoolThread: True
3. === After httpClient.GetStringAsync ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 32
     SynchronizationContext: 
     ManagedThreadId: 12
     IsThreadPoolThread: True
4. === After MethodWithoutConfigureAwaitFalseInsideAsync ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 1, Total threads: 32
     SynchronizationContext: 
     ManagedThreadId: 12
     IsThreadPoolThread: True
```

## 測試三：承測試二，如果不跑在IO相關的task，而跑在`Task.Delay`上，後續執行緒是？

### (3.1)不加`ConfigureAwait(false)`

#### (3.1)程式碼

```text
private async Task RunTest3_1()
{
    Debug.WriteLine("===== Test 3.1 =====");

    Debug.WriteLine("1. === Before Task.Delay ===");
    PrintInfos();

    await Task.Delay(1000);

    Debug.WriteLine("2. === After Task.Delay ===");
    PrintInfos();
}
```

#### (3.1)執行結果

##### (3.1)WPF

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 18
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 3.1 =====
1. === Before Task.Delay ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 18
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
2. === After Task.Delay ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 20
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
```

##### (3.1).NET Core Web API

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 1, Completion port threads: 0, Total threads: 25
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
===== Test 3.1 =====
1. === Before Task.Delay ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 25
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
2. === After Task.Delay ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 25
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
```

### (3.2)加`ConfigureAwait(false)`

#### (3.2)程式碼

```text
private async Task RunTest3_2()
{
    Debug.WriteLine("===== Test 3.2 =====");

    Debug.WriteLine("1. === Before Task.Delay ConfigureAwait(false) ===");
    PrintInfos();

    await Task.Delay(1000).ConfigureAwait(false);

    Debug.WriteLine("2. === After Task.Delay ConfigureAwait(false) ===");
    PrintInfos();
}
```

#### (3.2)執行結果

##### (3.2)WPF

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
===== Test 3.2 =====
1. === Before Task.Delay ConfigureAwait(false) ===
  -- Worker threads: 0, Completion port threads: 0, Total threads: 19
     SynchronizationContext: System.Windows.Threading.DispatcherSynchronizationContext
     ManagedThreadId: 1
     IsThreadPoolThread: False
2. === After Task.Delay ConfigureAwait(false) ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 21
     SynchronizationContext: 
     ManagedThreadId: 5
     IsThreadPoolThread: True
```

##### (3.2).NET Core Web API

```text
===== Current Thread Info (in TestStart method) =====
  -- Worker threads: 1, Completion port threads: 0, Total threads: 25
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
===== Test 3.2 =====
1. === Before Task.Delay ConfigureAwait(false) ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 25
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
2. === After Task.Delay ConfigureAwait(false) ===
  -- Worker threads: 1, Completion port threads: 0, Total threads: 25
     SynchronizationContext: 
     ManagedThreadId: 10
     IsThreadPoolThread: True
```

## 程式碼source code

[csharp-lab][]

[csharp-lab]: https://github.com/ThisWayne/csharp-lab
