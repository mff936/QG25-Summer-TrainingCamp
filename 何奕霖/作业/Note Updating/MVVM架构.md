# Compose

## 一、Compose布局

### **标准布局组件**：

* 使用Column可将多个项垂直地放在屏幕上
* 使用Row可将多个项水平地放置在屏幕上
* 使用Box可将一个元素放在另一个元素上
* Text：文本
* TextField：输入框
* TextButton：文字按钮

### **修饰符**：

修饰符地作用类似于基于视图地布局中的布局参数，借助修饰符可以修饰或扩充可组合项

* 更改可组合项地大小、布局、行为和外观
* 添加信息，如无障碍标签
* 处理用户输入
* 添加高级互动，如使元素可点击、可滚动、可拖动或可缩放

### **修饰符的顺序**：

由于每个函数都会对上一个函数返回的Modifier进行更改，因此**顺序会影响最终结果**（例如cliclable和padding之间的先后顺序）

### **Slots API**：

Material组件大量使用槽位API，这是Compose引入的一种模式，可在组合顶上带来一层自定义设置。这种方法使组件变得更加灵活，因为他们接受可以自行配置的子元素，而不必公开子元素的每个配置参数。**槽位会在界面中留出空白区域**。

### **Scaffold**：

Scaffold可以实现具有基本Material Design布局结构的界面。Scaffold可以为最常见的顶级Material组件（如TopAppBar、BottonAppBar、FloatingActionButton和Drawer）**提供槽位**。

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun LayoutStudy() {
    Scaffold(
        topBar = {
            TopAppBar(
                title =  {
                    Text(text = "LayoutStudy")
                },
                actions = {
                    IconButton(onClick = {}) {
                        Icon(imageVector = Icons.Filled.Favorite, contentDescription = null)
                    }
                }
            )
        }
    ){ innerPadding ->
        BodyContent(Modifier.padding(innerPadding))
    }
}

@Composable
fun BodyContent(modifier: Modifier = Modifier) {
    Column(modifier = modifier.padding(8.dp)) {
        Text(text = "Hello World!")
        Text(text = "Welcome to Jetpack Compose")
    }
}
```

### 使用列表：

* 如果用例不需要任何滚动，可以使用简单的Column或Row
* 如果需要显示大量列表项（或长度未知的列表），可以使用LazyColumn或LazyRow

```kotlin
//相当于ListView
@Composable
fun SimpleList() {
    val scrollState = rememberScrollState()	//记住滚动状态
    Column(Modifier.verticalScroll(scrollState)) {	//实现滚动
        repeat(100) {
            Text(text = "Item #$it", style = MaterialTheme.typography.bodyLarge)
        }
    }
}
```

```kotlin
//相当于RecyclerView，缓冲加载
@Composable
fun LazyList() {
    val scrollState = rememberLazyListState()
    LazyColumn(state = scrollState) {
        items(100) { index ->
            Text(text = "Item #$index", style = MaterialTheme.typography.bodyLarge)
        }
    }
}
```

只要控制的State发生改变页面就会重组（重绘）。

## 二、文字：

text：文字

color：文字颜色

fontSize：字体大小

fontStyle：字体样式，可设置为斜体Italic

fontWeight：字体权重，可设置字体加粗

overflow：文字溢出效果，与maxLines结合使用可实现文字溢出显示省略号效果

maxLines：最大行数，与overflow结合使用实现文字溢出显示省略号效果

### 单行文本多样式：

遇到一些情况，例如在一行文本中，其中某个地址可以点击，或者某个词语需要加粗，按照以往的操作，就需要使用多个TextView来实现或借助他人封装的View来实现。
在Compose中，text可以接收一个参数**AnnotatedString**

```kotlin
@Composable
fun ParagraphStyle() {
    Text(
        buildAnnotatedString {
            withStyle(style = ParagraphStyle(lineHeight = 30.sp)) {
                withStyle(style = SpanStyle(color = Color.Blue)) {
                    append("Hello")
                }
            }
        }
    )
}
```

buildAnnotatedString()方法里存在withStyle()和append()方法，这两个方法是AnnotatedString.Builder类里的方法；withStyke可以接收ParagraphStyle和SpanStyle，其中**ParagraphStyle是段落样式**，**SpanStyle是单行样式**。

于是便可以一直**套娃**，组合出想要的文本样式。

### 文本长按可选择：

Compose中提供了一个容器SelectionContainer，可以实现Text的选择效果

```kotlin
SelectionContainer {
    Text("This text is selectable")
}
```

### 可点击文本：

设置文本的点击事件可以使用Modifier提供的方法，点击会有水波涟漪的效果

```kotlin
Text(
    text = "Hello world",
    modifier = Modifier.clickable {
        //点击事件
    }
)
```

### 点击文本跳转链接：

```kotlin
@Composable
fun SelectableText() {
    val context = LocalContext.current //获取当前上下文
    val annotatedText = buildAnnotatedString {
        append("Click ")
        pushStringAnnotation(tag = "URL", annotation = "https://www.baidu.com")
        withStyle(style = SpanStyle(color = Color.Blue, fontWeight = FontWeight.Bold)) {
            append("here")
        }
        pop()
    }
    ClickableText(
        text = annotatedText,
        onClick = { offset ->
            annotatedText.getStringAnnotations(tag = "URL", start = offset, end = offset)
                .firstOrNull()?.let { annotation ->
                    //实际跳转逻辑
                    val intent = Intent(Intent.ACTION_VIEW, annotation.item.toUri())
                    context.startActivity(intent)
                }
        }
    )
}
```

## 三、图标Icon使用

