# 单线程模型
[[dart单线程模型.canvas|dart单线程模型]]

在Dart中，所有的外部事件任务都在事件队列中，如IO、计时器、点击、以及绘制事件等，而微任务通常来源于Dart内部，并且微任务非常少，之所以如此，是因为微任务队列优先级高，如果微任务太多，执行时间总和就越久，事件队列任务的延迟也就越久，对于GUI应用来说最直观的表现就是比较卡，所以必须得保证微任务队列不会太长。

值得注意的是，我们可以通过Future.microtask(…)方法向微任务队列插入一个任务。

在事件循环中，当某个任务发生异常并没有被捕获时，程序并不会退出，而直接导致的结果是当前任务的后续代码就不会被执行了，也就是说一个任务中的异常是不会影响其他任务执行的

# Flutter异常捕获
### Flutter框架异常捕获
Flutter 框架为我们在很多关键的方法进行了异常捕获。
```dart
@override
void performRebuild() {
 ...
  try {
    //执行build方法  
    built = build();
  } catch (e, stack) {
    // 有异常时则弹出错误提示  
    built = ErrorWidget.builder(_debugReportException('building $this', e, stack));
  } 
  ...
}      
```
发生异常时，Flutter默认的处理方式是弹一个ErrorWidget
```dart
FlutterErrorDetails _debugReportException(
  String context,
  dynamic exception,
  StackTrace stack, {
  InformationCollector informationCollector
}) {
  //构建错误详情对象  
  final FlutterErrorDetails details = FlutterErrorDetails(
    exception: exception,
    stack: stack,
    library: 'widgets library',
    context: context,
    informationCollector: informationCollector,
  );
  //报告错误 
  FlutterError.reportError(details);
  return details;
}
```
错误是通过FlutterError.reportError方法上报的
```dart
static void reportError(FlutterErrorDetails details) {
  ...
  if (onError != null)
    onError(details); //调用了onError回调
}
```
自己上报异常，只需要提供一个自定义的错误处理回调即可
```dart
void main() {
  FlutterError.onError = (FlutterErrorDetails details) {
    FlutterError.reportError(details);
  };
 ...
}
```
### 其他异常捕获与日志收集
在Flutter中，还有一些Flutter没有为我们捕获的异常，如调用空对象方法异常、Future中的异常。在Dart中，异常分两类：同步异常和异步异常，同步异常可以通过try/catch捕获,异步异常则使用runZoned (...) 方法.
###### runZoned
可以给执行对象指定一个Zone。Zone表示一个代码执行的环境范围,不同Zone之间是隔离的，沙箱可以捕获、拦截或修改一些代码行为，如Zone中可以捕获日志输出、Timer创建、微任务调度的行为，同时Zone也可以捕获所有未处理的异常。
```dart
R runZoned<R>(R body(), {
    Map zoneValues, 
    ZoneSpecification zoneSpecification,
}) 
```
zoneValues: Zone 的私有数据，可以通过实例`zone[key]`获取，可以理解为每个“沙箱”的私有数据。

zoneSpecification：Zone的一些配置，可以自定义一些代码行为，比如拦截日志输出和错误等，举个例子：
```dart
runZoned(
  () => runApp(MyApp()),
  zoneSpecification: ZoneSpecification(
    // 拦截print 蜀西湖
    print: (Zone self, ZoneDelegate parent, Zone zone, String line) {
      parent.print(zone, "Interceptor: $line");
    },
    // 拦截未处理的异步错误
    handleUncaughtError: (Zone self, ZoneDelegate parent, Zone zone,
                          Object error, StackTrace stackTrace) {
      parent.print(zone, '${error.toString()} $stackTrace');
    },
  ),
);
```
这样一来，我们 APP 中所有调用print方法输出日志的行为都会被拦截，通过这种方式，我们也可以在应用中记录日志，等到应用触发未捕获的异常时，将异常信息和日志统一上报。

另外我们还拦截了未被捕获的异步错误，这样一来，结合上面的 FlutterError.onError 我们就可以捕获我们Flutter应用错误了并进行上报了
```dart
import 'dart:async';

import 'package:flutter/material.dart';

void collectLog(String line) {
  // ... //收集日志
}
void reportErrorAndLog(FlutterErrorDetails details) {
  // ... //上报错误和日志逻辑
}

FlutterErrorDetails makeDetails(Object obj, StackTrace stack) {
  // 构建错误信息
  return FlutterErrorDetails(exception: obj, stack: stack);
}

void main() {
  var onError = FlutterError.onError; //先将 onerror 保存起来
  FlutterError.onError = (FlutterErrorDetails details) {
    onError?.call(details); //调用默认的onError
    reportErrorAndLog(details); //上报
  };
  runZoned(
    () => runApp(MyApp()),
    zoneSpecification: ZoneSpecification(
      // 拦截print
      print: (Zone self, ZoneDelegate parent, Zone zone, String line) {
        collectLog(line);
        parent.print(zone, "Interceptor: $line");
      },
      // 拦截未处理的异步错误
      handleUncaughtError: (Zone self, ZoneDelegate parent, Zone zone,
          Object error, StackTrace stackTrace) {
        FlutterErrorDetails details = makeDetails(error, stackTrace);
        reportErrorAndLog(details);
        parent.print(zone, '${error.toString()} $stackTrace');
      },
    ),
  );
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    throw UnimplementedError();
  }
}

```