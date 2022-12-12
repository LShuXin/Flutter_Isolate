# Flutter_Isolate

# 前言

我们编程是用多线程一般实现两种场景，一种是**异步执行**，一种是**并行执行**。

我们都知道 Dart 是单线程异步编程模型 这一点和js很像，它天生解决了异步执行的问题，详情查看[Flutter中的异步编程Future](https://www.psvmc.cn/article/2020-05-01-dart-future.html)。



但是并行执行怎么处理呢？在 Dart 中，它的线程概念被称为 `Isolate`。



它与我们之前理解的 Thread 概念有所不同，各个 isolate 之间是无法共享内存空间，isolate 之间有自己的 event loop。我们只能通过 Port 传递消息，然后在另一个 isolate 中处理然后将结果传递回来，这样我们的 UI 线程就有更多余力处理 pipeline，而不会被卡住。

将非常耗时的任务添加到事件队列后，会拖慢整个事件循环的处理，甚至是阻塞。可见基于事件循环的异步模型仍然是有很大缺点的，这时候我们就需要`Isolate`，这个单词的中文意思是隔离。

简单说，可以把它理解为Dart中的线程。但它又不同于线程，更恰当的说应该是微线程。**它与线程最大的区别就是不能共享内存，因此也不存在锁竞争问题**，两个`Isolate`完全是两条独立的执行线，且每个`Isolate`都有自己的**事件循环**，它们之间只能通过发送消息通信，所以它的资源开销低于线程。



> 下文中所说的Dart中的线程都是指的微线程。 



所以说Isolate，一句话总结它的作用就是

> Isolate可以实现异步并行多个任务
>
> Future实现异步串行多个任务 



# 使用场景

在 Dart 中 async 和 Future   无法解决所有耗时的工作。Dart 虽然支持异步执行，但其实如果是通过 async 的话，只是把工作丟到同一个 event loop 中， 让他暂时不会卡住目前的工作 ， 等到真的轮到它执行的时候 ，如果它真的很耗时，那 main **isolate** 还是会 freeze(冻结) 住的  (为什么会冻结？ 主线程负责 UI的渲染工作. 但是如果密集型计算很耗时, 假如这个计算占用1s的时间, 你的UI就会卡住1s) 。Dart 主要的 task 都是在 main **isolate** 中完成的，**isolate** 像是个 single thread 的 process。如果真的想要让某些工作能夠同时执行，不要卡住 main **isolate** 的话，就得要自己产生新的 isolate 來执行。

`Isolate`虽好，但也有合适的使用场景，不建议滥用`Isolate`，应尽可能多的使用Dart中的事件循环机制去处理异步任务，这样才能更好的发挥Dart语言的优势。

在Dart中我们使用多线程计算的时候，整个计算的时间会比单线程还要多，额外的耗时是什么呢？

- 创建 Isolate
- 线程间传递数据

**Isolate 实际上是比较重的，每当我们创建出来一个新的 Isolate 至少需要 2mb 左右的空间甚至更多，取决于我们具体 isolate 的用途。**

那么应该在什么时候使用**Future**，什么时候使用**Isolate**呢？



一个最简单的判断方法是根据某些任务的平均时间来选择：

- 方法执行在几毫秒或十几毫秒左右的，应使用`Future`

- 如果一个任务需要几百毫秒或之上的，则建议创建单独的`Isolate`

  

一些常见的可以参考的场景

- JSON 解码

- 加密

- [图像处理](https://cloud.tencent.com/product/tiia?from=10680)：比如剪裁

- 网络请求：加载资源、图片

  

# 创建Isolate

`Isolate`由一对Port分别由用于接收消息的`ReceivePort`对象，和用于发送消息的`SendPort`对象构成。其中`SendPort`对象不用单独创建，它已经包含在`ReceivePort`对象之中。需要注意，一对Port对象只能单向发消息，这就如同一根自来水管，`ReceivePort`和`SendPort`分别位于水管的两头，水流只能从`SendPort`这头流向`ReceivePort`这头。因此，两个`Isolate`之间的消息通信肯定是需要两根这样的水管的，这就需要两对Port对象。

## Dart中创建

```javascript
import 'dart:isolate';
import  'dart:io';

void main() {
  print('main isolate start');
  create_isolate();
  print('main isolate end');
}

// 创建一个新的 isolate
create_isolate() async {
  ReceivePort rp = new ReceivePort();
  SendPort port1 = rp.sendPort;

  Isolate newIsolate = await Isolate.spawn(doWork, port1);

  SendPort port2;
  rp.listen((message) {
    print('main isolate message: $message');
    if (message[0] == 0) {
      port2 = message[1];
    } else {
      port2?.send([1, '这条信息是 main isolate 发送的']);
    }
  });
}

// 处理耗时任务(static!!!)
static void doWork(SendPort port1) {
  print('new isolate start');
  ReceivePort rp2 = new ReceivePort();
  SendPort port2 = rp2.sendPort;

  rp2.listen((message) {
    print('doWork message: $message');
  });

  // 将新isolate中创建的SendPort发送到主isolate中用于通信
  port1.send([0, port2]);
  // 模拟耗时5秒
  sleep(Duration(seconds:5));
  port1.send([1, 'doWork 任务完成']);

  print('new isolate end');
}
```



运行结果

>  main isolate start main isolate end new isolate start main isolate message: [0, SendPort] new isolate end main isolate message: [1, doWork 任务完成] doWork message: [1, 这条信息是 main isolate 发送的] 



运行后都会创建两个进程，一个是主`Isolate`的微进程，一个是新`Isolate`的微进程，两个微进程都双向绑定了消息通信的通道，即使新的`Isolate`中的任务完成了，它的微进程也不会立刻退出，因此，当使用完自己创建的`Isolate`后，最好调用`newIsolate.kill(priority: Isolate.immediate);`将`Isolate`立即杀死。



## Flutter中创建

在Dart中创建一个`Isolate`显得有些繁琐，可惜的是Dart官方并未提供更高级封装。

但是，如果想在Flutter中创建`Isolate`，则有更简便的API，这是由`Flutter`官方进一步封装`ReceivePort`而提供的更简洁API。

使用`compute`函数来创建新的`Isolate`并执行耗时任务

```javascript
import 'package:flutter/foundation.dart';
import  'dart:io';

// 创建一个新的Isolate，在其中运行任务doWork
create_new_task() async {
  var str = 'New Task';
  var result = await compute(doWork, str);
  print(result);
}

// static 也是关键
static String doWork(String value) {
  print('new isolate doWork start');
  // 模拟耗时5秒
  sleep(Duration(seconds:5));

  print('new isolate doWork end');
  return 'complete:$value';
}
```



`compute`函数有两个必须的参数，第一个是待执行的函数，这个函数必须是一个顶级函数或静态方法，不能是类的实例方法，第二个参数为动态的消息类型，可以是被运行函数的参数。

需要注意，使用`compute`应导入`'package:flutter/foundation.dart'`包。



# 线程管理器

```javascript
import 'dart:isolate';
typedef LikeCallback = void Function(Object value);

class ThreadManagement {
  //entryPoint 必须是静态方法
  static Future<Map> runtask(void entryPoint(SendPort message), LikeCallback(Object value), {Object parameter}) async {
    final response = ReceivePort();
    Isolate d = await Isolate.spawn(entryPoint, response.sendPort);
    
    // 调用sendReceive自定义方法
    if (parameter != null) {
      SendPort sendPort = await response.first;
      ReceivePort receivePort = ReceivePort();
      sendPort.send([parameter, receivePort.sendPort]);
      receivePort.listen((value)　{
        receivePort.close();
        d.kill();
        LikeCallback(value);
      });
    
      return {
        'isolate': d,
        'receivePort': receivePort,
      };
    } else {
      response.listen((value) {
        response.close();
        d.kill();
        LikeCallback(value);
      });

      return {
        'isolate': d,
        'receivePort': response,
      };
    }
  }
}
```



## 调用无参任务

不带参数任务

```javascript
static void getbannerthread(SendPort port) async {
  var c = await HttpManager.isolationnetFetch(
    'http://www.psvmc.cn/getbanner',
    null,
    null, 
    new Options(method: 'GET')
  );
  port.send(c);  // 是回调计算结果
}
```



调用任务

```javascript
ThreadManagement.runtask(API.getbannerthread, (value) {
  if (value != null) {
    //业务逻辑
  }
});
```



## 调用有参任务

带参数任务

```javascript
static getVideolisttask(SendPort port) async {
  ReceivePort receivePort = ReceivePort();
  port.send(receivePort.sendPort);
  
  // 监听外界调用(也可以用listen)
  await for (var msg in receivePort) {
    Map requestURL = msg[0];
    SendPort callbackPort = msg[1];
    receivePort.close();

    var res = await HttpManager.isolationnetFetch(
      'http://xxxxxx?type=' + requestURL['type'] + '&after=' + requestURL['after'],
      null,
      null,
      new Options(method: 'GET')
    );

    callbackPort.send(res);
  }
}
```



执行带参数的任务

```javascript
ThreadManagement.runtask(API.getVideolisttask, (value) {
  if(value != null){
    //业务逻辑
  }
}, parameter: {
  'type':'hot',
  'after':'1'
});
```



# 线程池

使用LoadBalancer

如何减少 isolate 创建所带来的消耗呢。自然一个想法就是能否创建一个线程池，初始化到那里。当我们需要使用的时候再拿来用就好了。

实际上 dart team 已经为我们写好一个非常实用的 package，其中就包括 `LoadBalancer`。

我们现在 `pubspec.yaml` 中添加 isolate 的依赖。

```javascript
isolate: ^2.0.2
```



然后我们可以通过 `LoadBalancer` 创建出指定个数的 isolate。

由于 dart 天生支持顶层函数，我们可以在 dart 文件中直接创建这个 `LoadBalancer`。

```javascript
Future<LoadBalancer> loadBalancer = LoadBalancer.create(2, IsolateRunner.spawn);
```



这段代码将会创建出一个 isolate 线程池，并自动实现了[负载均衡](https://cloud.tencent.com/product/clb?from=10680)。

下面我们再来看看应该如何使用 `LoadBalancer` 中的 isolate。

```javascript
int useLoadBalancer() async {
   final lb = await loadBalancer;
   int res = await lb.run<int, int>(_doSomething, 1);
   return res;
 }
```



我们关注的只有 `Future<R> run<R, P>(FutureOr<R> function(P argument), argument,` 方法。我们还是需要传入一个 `function` 在某个 isolate 中运行，并传入其参数 `argument`。run 方法将会返回我们执行方法的返回值。

整体和 compute 使用感觉上差不多，但是当我们多次使用额外的 isolate 的时候，不再需要重复创建了。

并且 `LoadBalancer` 还支持 runMultiple，可以让一个方法在多线程中执行。

`LoadBalancer` 经过测试，它会在第一次使用其 isolate 的时候初始化线程池。

当应用打开后，即使我们在顶层函数中调用了 `LoadBalancer.create`，但是还是只会有一个 Isolate。

当我们调用 `run` 方法时，才真正创建出了实际的 isolate。