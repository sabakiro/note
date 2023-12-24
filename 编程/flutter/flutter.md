# 示例分析
```dart
//导入了 Material UI 组件库
import 'package:flutter/material.dart';
//应用入口
void main() => runApp(MyApp());
//无状态组件
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(title: 'Flutter Demo Home Page'),
    );
  }
}
/*
状态组件
Stateful widget 可以拥有状态，这些状态在 widget 生命周期中是可以变的，
而 Stateless widget 是不可变的。

Stateful widget 至少由两个类组成：

1. 一个StatefulWidget类。
2. 一个 State类； StatefulWidget类本身是不变的，但是State类中持有的状态在 widget 生命周期中可能会发生变化。
*/
class MyHomePage extends StatefulWidget {
  MyHomePage({Key? key, required this.title}) : super(key: key);
  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
	//组件状态
  int _counter = 0;
	/*
	设置状态的自增函数
	先自增_counter，然后调用setState 方法。
	setState方法的作用是通知 Flutter 框架，
	有状态发生了改变，Flutter 框架收到通知后，
	会执行 build 方法来根据新的状态重新构建界面
  */
	void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text('You have pushed the button this many times:'),
            Text(
              '$_counter',
              style: Theme.of(context).textTheme.headline4,
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
      //接受一个回调函数，代表它被点击后的处理器
        onPressed: _incrementCounter,
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ), 
    );
  }
}
```

# Flutter 框架的处理流程：
1. 根据 Widget 树生成一个 Element 树，Element 树中的节点都继承自 Element 类。
2. 根据 Element 树生成 Render 树（渲染树），渲染树中的节点都继承自RenderObject 类。
3. 根据渲染树生成 Layer 树，然后上屏显示，Layer 树中的节点都继承自 Layer 类。


# 示例
```dart
Container( // 一个容器 widget
  color: Colors.blue, // 设置容器背景色
  child: Row( // 可以将子widget沿水平方向排列
    children: [
      Image.network('https://www.example.com/1.png'), // 显示图片的 widget
      const Text('A'),
    ],
  ),
);
/*
注意，如果 Container 设置了背景色，
Container 内部会创建一个新的 ColoredBox 来填充背景
if (color != null)
  current = ColoredBox(color: color!, child: current);
*/
```
而 Image 内部会通过 RawImage 来渲染图片、Text 内部会通过 RichText 来渲染文本
Widget
Container
	ColoredBox
		Row
			Image
				RawImage
			Text
				RichText

Element Tree
ComponentElement
	RenderObjectElement
		RenderObjectElement
			Component
				RenderObjectElement
			Component
				RenderObjectElement

Render Tree
RenderDecoratedBox
	RenderFlex
		RenderImage
		RenderParagraph

三棵树中，Widget 和 Element 是一一对应的，但并不和 RenderObject 一一对应。

[[0.Widget]]

[[0.组件管理]]
