---
title: 'C#的Async & Await筆記'
date: 2019-08-23 17:51:46
urlname: csharp-async-await-note
tags:
  - C#
  - async&await
---

最一開始學習C#的時候，搞不太清楚`async`/`await`實際上到底發生什麼事，有些人認為用`await`就是新增一個執行緒去執行一個`async method`，用多執行緒來平行處理的概念，但是又看到一般建議如果有用到`async`/`await`，最好就是all the way都用`async`/`await`，不要中間又用`Task.Result`等等不是`await`的方式，如果真的每寫一個`await`就是新增一條執行緒去執行，那all the way都是`await`不就占用超多條執行緒？

當時沒有辦法有其他方式理解`async`/`await`，只能照著建議的使用方式去寫程式，想深入了解`async`/`await`但常常看不懂，看文件上寫CPU bound與IO bound的情況，沒有辦法真的理解情況的不同會怎麼影響執行與效能，看到文件上寫碰到`await`會跳回到呼叫端，等`await`裡面的東西結束了就會從`await`下方自動繼續往下執行，腦袋裡的模型暫時只能想像常常都是有一個執行緒被拿去執行`await`裡面的東西。

之後念了一些作業系統的概念，再查一次資料，回來重新理解`async`/`await`，發現以前腦袋裡那些暫時先那樣想的模型，有些可以說是對的也可以說不完全對，`async`/`await`的題目有很多東西可以研究，在這邊來做一點筆記，有錯誤還請指正。

<!-- more -->

## TL;DR，細節懶得看只想先知道怎麼做

下一篇[C#的Async & Await筆記(2)](../csharp-async-await-note-2/)有程式碼可以玩玩看

1. 平行執行`Task`

   多個`MethodAsync`呼叫可以先持有各method回傳的`Task`，做其他事情，然後真正需要倚賴`Task`的結果時再用`await`依據結果繼續執行。

2. 不要用`async void`

3. 沒真的需要用`await`，可以不用`return await MethodAsync(...)`，直接傳回給呼叫端`Task`

4. 寫CPU bound的library API，讓呼叫方自行決定怎麼控制執行緒來執行
5. 寫library除非一定需要跟UI thread互動，否則建議使用`await`時加上`ConfigureAwait(false)`
6. 非同步處理是一個目的，多執行緒是一種達成方式，但不是唯一的達成方式。用`await`執行IO bound的method，IO等待期間沒有執行緒被占用著等待IO，`async`/`await`本身也不額外新增執行緒

## 1. 平行執行`Task`

當持有回傳的`Task`時，實際上`Task`的內容已經開始執行。
可以的話先開始執行耗時的`async method`，再執行其他不倚賴剛才`async method`結果的運算，一定需要`async method`的結果時再用`await`。

```csharp
// C#
// 下面這一行會建立並開始一個Task
var myTask = someWebAccessMethodAsync(url);

// 當Task執行期間，可以先做其他不依賴Task結果的事情...

// 當前method會停止並跳回給呼叫端，await之後的程式會在Task完成以後繼續被執行
var result = await myTask;  
```

有多個`async method`先取得各個`Task`，如此各個`async method`即開始處理，在需要結果的時候再用`await`。

```csharp
// C#
Task<int> download1 =
    ProcessURLAsync("https://msdn.microsoft.com", client);

Task<int> download2 =
    ProcessURLAsync("https://msdn.microsoft.com/library/hh156528(VS.110).aspx", client);  

Task<int> download3 =
    ProcessURLAsync("https://msdn.microsoft.com/library/67w7t67f.aspx", client);  

// 需要個別Task的結果再依序用await
int length1 = await download1;  
int length2 = await download2;  
int length3 = await download3;
```

參考  
[How to: Make Multiple Web Requests in Parallel by Using async and await (C#)][]

## 2. 不要用`async void`

除非是最上層的event handler需要，否則不要用`async void`。

呼叫端沒辦法知道`async method`什麼時候工作結束，可能導致race condition，下面案例line A和line B被執行到的順序不一定，有可能經過了2秒line B還沒完成，line A就先往下印出`m_GetResponse`。

```csharp
// C#
private async void Button1_Click(object Sender, EventArgs e) {
    try {
        SendData("https://secure.flickr.com/services/oauth/request_token");
        await Task.Delay(2000);// line A, race condition with line B
        DebugPrint("Received Data: " + m_GetResponse);
    }
    catch (Exception ex) {
        rootPage.NotifyUser("Error posting data to server." + ex.Message);
    }
}

private async void SendData(string Url) {
    var request = WebRequest.Create(Url);
    using (var response = await request.GetResponseAsync()) // line B, race condition with line A
    using (var stream = new StreamReader(response.GetResponseStream()))
        m_GetResponse = stream.ReadToEnd();
}
```

`async void`會fire and forget，預期的`try`/`catch`會抓不到exception。

```csharp
// C#
private async void Button1_Click(object Sender, EventArgs e) {
    try {
        SendData("https://secure.flickr.com/services/oauth/request_token");
        //await Task.Delay(2000);
        //DebugPrint("Received Data: " + m_GetResponse);
    }
    catch (Exception ex) {
        // 有可能抓不到SendData丟出來的exception
        // 因為等SendData裡面的await後續開始執行時
        // main thread可能已經執行完整個Button1_Click
        // 最後exception可能會是更外層才接到然後跳到螢幕上顯示
        rootPage.NotifyUser("Error posting data to server." + ex.Message);
    }
}

private async void SendData(string Url) {
    var request = WebRequest.Create(Url);
    using (var response = await request.GetResponseAsync())
    using (var stream = new StreamReader(response.GetResponseStream()))
        m_GetResponse = stream.ReadToEnd();
}
```

上述程式比較好的寫法

```csharp
// C#
private async void Button1_Click(object Sender, EventArgs e) {
    try {
        m_GetResponse = await SendDataAsync("https://secure.flickr.com/services/oauth/request_token");

        DebugPrint("Received Data: " + m_GetResponse);
    }
    catch (Exception ex) {
        rootPage.NotifyUser("Error posting data to server." + ex.Message);
    }
}

private async Task<string> SendDataAsync(string Url) {
    var request = WebRequest.Create(Url);
    using (var response = await request.GetResponseAsync())
    using (var stream = new StreamReader(response.GetResponseStream()))
        return stream.ReadToEnd();
}
```

參考  
[Tip 1: Async void is for top-level event-handlers only][]

## 3. 沒真的需要用`await`，可以不用`return await MethodAsync(...)`，直接傳回給呼叫端`Task`

```csharp
// C#
// 多餘的async和await
public async Task<string> Method(...) {
    // 中間做一些不用await的事
    return await IORequestAsync(...);
}
// 不用掛上async，直接回傳Task就可以了
public Task<string> Method(...) {
    return IORequestAsync(...);
}
```

## 4. 寫CPU bound的library API，讓呼叫方自行決定怎麼控制執行緒來執行

比方說要寫的API需要做大量Deserialize，或是很多需要計算的for迴圈，CPU bound，這時候讓呼叫API的地方自行決定該怎麼控制多少執行緒來執行，不要包起來默默的占用執行緒。

參考  
[Tip 2: Distinguish CPU-Bound work from IO-bound work][]

## 5. 寫library除非需要跟UI thread互動，否則建議使用`await`時加上`ConfigureAwait(false)`

一般有UI thread的應用程式，因為使用SynchronizationContext機制的原因，所以在`await`的`Task`完成之後，機制上會把剩下要執行的部分用`SynchroniztionContext.Post`的方式（把要做的事情包起來丟到一個queue，UI thread會去把這個queue裡的工作做掉）丟給UI thread去執行，容易導致UI thread執行太多不必要的工作，導致UI thread被佔用畫面卡住，所以建議用`await`的地方加上`ConfigureAwait(false)`讓`await`的`Task`完成後，不要透過SynchronizationContext的機制執行後續的程式，而透過執行緒池裡的執行緒完成。

## 6. 非同步是一個目的，多執行緒是一種達成方式，但不是唯一的達成方式。用`await`執行IO bound的method，IO等待期間沒有執行緒被占用著等待IO，`async`/`await`本身也不額外新增執行緒

先知道兩點：

1. 一般硬體／作業系統有機制可以不用只靠執行緒一直問IO是否完成了來知道IO狀態，而是當IO完成了來通知process該IO已經完成了。

2. thread pool裡面有`worker thread`、`IOCP thread`兩種。

從上層一點來說，當呼叫了一個IO操作，會取得一個`Task`，此時IO已經開始執行，當執行到`await task`，如果`Task`還沒完成，當前執行緒會跳回到caller端，如果`Task`已經完成，則會直接繼續執行。

`await`後續還未完成的部分，會在實際IO結束完成後，透過IOCP(Input Output Completion Port)的機制，由`IOCP thread`來接手，如果是有SynchronizationContext(通常有UI的像是WinForm, WPF都有)，會以`SynchroniztionContext.Post`的方式丟給UI thread去執行`await`後續的程式，如果有加上`ConfigureAwait(false)`則由當前`IOCP thread`繼續執行。

compiler在compile的時候看到`await`實際上有做一些手腳，程式執行到`await`這邊會先去看`Task`是不是已經完成了（fast path優化），如果已經完成了則沒必要透過額外的`await`機制增加負擔，而是直接繼續執行就可以了，所以程式碼上`await`後續執行的thread也是有可能是原先的thread，還有些情況像是做測試就可以用`Task.FromResult`直接給一個完成了的`Task`可以省去額外的`await`負擔。

非同步是一個目的，多執行緒是一種達成方式，但不是唯一的達成方式。過程中主執行緒在IO等待期間沒有閒置也一直繼續執行不倚賴IO結果的程式，而當IO完成了，`await`後續的程式是由thread pool裡面的`IOCP thread`來接手，沒有執行緒空等，過程中沒有額外的新增執行緒，全靠原本thread pool的機制在管理執行緒數量。不依賴IO結果的程式與等待IO兩件工作非同步的在進行。

參考  
[Lucian Wischik - Async Part 1 — new feature in Visual Studio 11 for responsive programming.][]  
[There Is No Thread][]

## 小記IOCP

一個專門用來處理同時有多個IO的Asynchronous I/O機制，裡面主要有一個`I/O Completion Queue`以FIFO存放完成了的IO工作相關資訊（包含IO本身資訊，以及要繼續執行的程式），有一個`WaitingThread List`以LIFO存放執行緒（一個執行緒執行完一個工作要再執行下一個工作，LIFO有機會因省去context switch而速度加快）。

應用端呼叫IO時附上操作IO的資訊以及IO完成後要怎麼處理的資訊，當IO完成後這些資訊就會被放到`I/O Completion Queue`，然後會有一定數量的`IOCP thread`去處理這些完成了的IO工作。以往常常是一個thread來處理一個IO，有很多個IO request就要很多個thread，而現在是多個thread去檢查一個存放完成了IO的queue，如果沒有要處理的IO在`I/O Completion Queue`裡面，thread就會被block住放進`WaitingThread List`，等`I/O Completion Queue`又有IO完成了的資訊，再以LIFO的方式從`WaitingThread List`取出thread喚醒來工作。

參考  
[Asynchronous I/O][]  
[IOCP][]  
[工作者线程（worker thread）和I/O线程][]  
[I/O Completion Ports][]

## 全部參考

C# async & await，我覺得這些資料都解釋的很好很值得一讀：

1. [How to: Make Multiple Web Requests in Parallel by Using async and await (C#)][]
2. [Six Essential Tips for Async][]
3. [Tip 1: Async void is for top-level event-handlers only][]
4. [Tip 2: Distinguish CPU-Bound work from IO-bound work][]
5. [Lucian Wischik - Async Part 1 — new feature in Visual Studio 11 for responsive programming.][]
6. [Lucian Wischik - Async Part 2 — deep dive into the new language feature of VB/C#][]
7. [談C# 編譯器編譯前的程式碼擴展行為 (2017年續　上)][]

IOCP相關：

1. [Asynchronous I/O][]
2. [IOCP][]
3. [工作者线程（worker thread）和I/O线程][]
4. [I/O Completion Ports][]

[Lucian Wischik - Async Part 1 — new feature in Visual Studio 11 for responsive programming.]: https://vimeo.com/43808831

[Lucian Wischik - Async Part 2 — deep dive into the new language feature of VB/C#]: https://vimeo.com/43808833

[How to: Make Multiple Web Requests in Parallel by Using async and await (C#)]: https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/how-to-make-multiple-web-requests-in-parallel-by-using-async-and-await

[Six Essential Tips for Async]: https://channel9.msdn.com/Series/Three-Essential-Tips-for-Async

[Tip 1: Async void is for top-level event-handlers only]: https://channel9.msdn.com/Series/Three-Essential-Tips-for-Async/Tip-1-Async-void-is-for-top-level-event-handlers-only

[Tip 2: Distinguish CPU-Bound work from IO-bound work]: https://channel9.msdn.com/Series/Three-Essential-Tips-for-Async/Tip-2-Distinguish-CPU-Bound-work-from-IO-bound-work

[There Is No Thread]: https://blog.stephencleary.com/2013/11/there-is-no-thread.html

[談C# 編譯器編譯前的程式碼擴展行為 (2017年續　上)]: https://dotblogs.com.tw/code6421/2017/01/24/csharp5

[Asynchronous I/O]: https://en.wikipedia.org/wiki/Asynchronous_I/O
[IOCP]: https://zh.wikipedia.org/wiki/IOCP
[工作者线程（worker thread）和I/O线程]: https://blog.51cto.com/cnn237111/1437475
[I/O Completion Ports]: https://docs.microsoft.com/zh-tw/windows/win32/fileio/i-o-completion-ports
