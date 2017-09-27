---
title: Java 项目优化实战
date: 2016-03-27 21:55
---

## 1 Visual VM

项目中的某一个接口，在某一场景下（数据量大），性能让人难以忍受。

那么如何有什么工具可以定位引发性能问题的代码呢？其实有很多，这里我们使用 Visual VM。

{% asset_img bg2016031203.png %}

Visual VM 是一款用来分析 Java 应用的图形工具，能够对 Java 应用程序做性能分析和调优。如果你使用的 java 7 或者 java 8，那么可以直接在 JDK 的 bin 目录找到该工具，名称为 jvisualvm。当然也可以在[官网](http://visualvm.java.net/)上自行下载。

使用 Visual VM 分析某个接口的性能的方法如下：

{% asset_img bg2016031204.png %}

结果显示如下:

{% asset_img bg2016031201.png %}

通过上图，我们可以看到比较耗时的方法为 resolveBytePosition 和 rest，getFile 和 currentUser 是网络请求，暂不考虑。

## 2 优化一

### 2.1 背景

首先拿 resolveBytePosition 方法开刀。为了能更容易的解释 resolveBytePosition 的用途，举个例子。

给定一个字符串 chars 与该字符串的 UTF-8 二进制数组(空格用来隔开字符数据，实际并不存在):

    chars = "just一个test";
    bytes = "6A 75 73 74 E4B880 E4B8AA 74 65 73 74";

resolveBytePosition 用来解决给定一个 bytes 的偏移 bytePos 计算 chars 中的偏移 charPos 的问题。比如:

    bytePos = 0 (6A) 对应 charPos = 0 (j)
    bytePos = 1 (75) 对应 charPos = 1 (u)


如果使用 array[start:] 表示从下标 start 开始截取数组元素至末尾组成的新数组，那么则有：

    bytes[bytePos:] = chars[charPos:]

举例:

    bytes[0:] = chars[0:]
    bytes[1:] = chars[1:]
    bytes[10:] = chars[6:]

### 2.2 原实现

明白了 resolveBytePosition 的作用，看一下它的实现

```
public int resolveBytePosition(byte[] bytes, int bytePos) {
    return new String(slice(bytes, 0, bytePos)).length();
}
```

该解法简单粗暴，能够准确的计算出结果，但是缺点显而易见，频繁的构建字符串，对性能造成了极大的影响。通过 Visual VM 可以证实我们的推论，通过点击快照，查看更详细的方法调用耗时。

{% asset_img bg2016031205.png %}

### 2.3 剖析

为了更方便的剖析问题，我们绘制如下表格，用来展示每一个字符的 UTF-8 以及 Unicode 的二进制数据：

|| j | u | s |t | 一 |个 | t|e|s|t|
|:--:| :--:  | :--: |:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|UTF-8| 6A | 75 |73|74|E4B880|E4B8AA|74|65|73|74|
|Unicode| 6A | 75 |73|74|4E00|4E2A|74|65|73|74|



接着我们将字节数据转换为字节长度：

|| j | u | s |t | 一 |个 | t|e|s|t|
|:--:| :--:  | :--: |:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|UTF-8| 1 | 1 |1|1|3|3|1|1|1|1|
|Unicode| 1 | 1 |1|1|2|2|1|1|1|1|

Java中的使用 char 来表示Unicode，char 的长度为 2 个字节，因此一个 char 足以表示示例中的任何一个字符。

我们使用一个单元格表示一个byte（UTF-8）或一个char（Unicode），并对单元格编号，得到下表：

<table><thead><tr><th style="text-align: center">&nbsp;</th><th style="text-align: center">j</th><th style="text-align: center">u</th><th style="text-align: center">s</th><th style="text-align: center">t</th><th style="text-align: center" colspan="3">一</th><th style="text-align: center" colspan="3">个</th><th style="text-align: center">t</th><th style="text-align: center">e</th><th style="text-align: center">s</th><th style="text-align: center">t</th></tr></thead><tbody><tr><tr><td style="text-align: center">bytes</td><td style="text-align: center">0</td><td style="text-align: center">1</td><td style="text-align: center">2</td><td style="text-align: center">3</td><td style="text-align: center">4</td><td style="text-align: center">5</td><td style="text-align: center">6</td><td style="text-align: center">7</td><td style="text-align: center">8</td><td style="text-align: center">9</td><td
style="text-align: center">10</td><td
style="text-align: center">11</td><td style="text-align: center">12</td><td style="text-align: center">13</td></tr><td style="text-align: center">chars</td><td style="text-align: center">0</td><td style="text-align: center">1</td><td style="text-align: center">2</td><td style="text-align: center">3</td><td style="text-align: center" colspan="3">4</td><td style="text-align: center" colspan="3">5</td><td style="text-align: center">6</td><td style="text-align: center">7</td><td style="text-align: center">8</td><td style="text-align: center">9</td></tr></tbody></table>

可以得出下面对应关系

    bytes[0:] = chars[0:]
    bytes[1:] = chars[1:]
    bytes[2:] = chars[2:]
    bytes[3:] = chars[3:]
    bytes[4:] = chars[4:]
    bytes[7:] = chars[5:]
    bytes[10:] = chars[6:]
    ... ...

### 2.3 方案

进行到这一步，高效的算法已经呼之欲出了。算法如下：

把字符 UTF-8 数据的二进制长度不为 1 的称为特征点。除特征点外，每个字符都是一个字节长度。记下所有特征点的对应关系，对于给定的 bytePos，都可以根据公式计算得到 charPos。

公式为:

    charPos = bytePos - preBytePos + preCharPos

举例：

则本实例中有两个特征点 `一`、`个`，记作：

    bytes[6:] = chars[4:]
    bytes[9:] = chars[5:]

如果给定 bytePos 10, 首先找到前一个特征点的对应关系 9（preBytePos） -> 5（preCharPos), 根据公式得出 (10 - 9) + 5 = 6。

### 2.4 核心代码

该算法还有一个比较关键的问题要解决，即高效的计算一个 char 的字节长度。计算 char 的字节长度的算法参考了 [StackOverflow](http://stackoverflow.com/questions/2726071/efficient-way-to-calculate-byte-length-of-a-character-depending-on-the-encoding)。

```
// 计算特征点
private int[][] calcSpecialPos(String str) {
    ArrayList<int[]> specialPos = new ArrayList<>()

    specialPos.add(new int[] {0, 0});

    int lastCharPost = 0;
    int lastBytePos = 0;

    Charset utf8 = Charset.forName("UTF-8");
    CharsetEncoder encoder = utf8.newEncoder();
    CharBuffer input = CharBuffer.wrap(str.toCharArray());
    ByteBuffer output = ByteBuffer.allocate(10);

    int limit = input.limit();
    while(input.position() < limit) {
        output.clear();
        input.mark();
        input.limit(Math.min(input.position() + 2, input.capacity()));
        if (Character.isHighSurrogate(input.get()) && !Character.isLowSurrogate(input.get())) {
            //Malformed surrogate pair
            lastCharPost++;
        }
        input.limit(input.position());
        input.reset();
        encoder.encode(input, output, false);

        int encodedLen = output.position();
        lastCharPost++;
        lastBytePos += encodedLen;

        if (encodedLen != 1) { // 特征点
            specialPos.add(new int[]{lastBytePos, lastCharPost});
        }
    }


    return toArray(specialPos);
}

// 根据特征点，计算 bytePos 对应的 charPos
private int calcPos(int[][] specialPos, int bytePos) {
    // 如果只有一个元素 {0, 0)，说明没有特征值
    if (specialPos.length == 1) return bytePos;

    int pos = Arrays.binarySearch(specialPos,
            new int[] {bytePos, 0},
            (int[] a, int[] b) -> Integer.compare(a[0], b[0]));

    if (pos >= 0) {
        return specialPos[pos][1];
    } else {
        // if binary search not fonund, will return (-(insertion point) - 1),
        // so here -2 is mean -1 to get insertpoint and then -1 to get previous specialPos
        int[] preSpecialPos = specialPos[-pos-2];
        return bytePos - preSpecialPos[0] + preSpecialPos[1];
    }
}
```


## 3 优化二

### 3.1 背景

接下来解决第二个函数 rest。该函数的功能是得到 JsonArray（gson） 的除第一个元素外的所有元素。

由于 rest 是在一个递归函数中被调用且递归栈很深，因此如果 rest 实现的不够高效，其影响会被成倍放大。

### 3.2 原实现

```
private JsonArray rest(JsonArray arr) {
    JsonArray result = new JsonArray();
    if (arr.size() > 1) {
        for (int i = 1; i < arr.size(); i++) {
            result.add(arr.get(i));
        }
    }
    return result;
}
```

### 3.3 剖析

通过调试发现 JsonArray 中存储了相当大的数据，对于频繁调用的场景，每次都对其重新构建明显不是一个明智的选择。
通过查看返回的 JsonArray 使用情况，我们得到了另一条线索：仅仅使用里面的数据，而不涉及修改。

考虑到 JsonArray 被实现成 final，最后方案确定为实现一个针对 rest 这种需求定制的代理类。

### 3.4 方案 & 代码


代理类 JsonArrayWrapper 分别对 first、rest、foreach 等功能进行了实现。

```
class JsonArrayWrapper implements Iterable<JsonElement> {
    private JsonArray jsonArray;

    private int mark;

    public JsonArrayWrapper() {
        this.jsonArray = new JsonArray();
        this.mark = 0;
    }

    public JsonArrayWrapper(JsonArray jsonArray) {
        this.jsonArray= jsonArray;
        this.mark = 0;
    }

    public JsonArrayWrapper(JsonArray jsonArray, int mark) {
        this.jsonArray = jsonArray;
        this.mark = mark;
    }

    public JsonObject first() {
        return jsonArray.get(mark).getAsJsonObject();
    }

    public JsonArrayWrapper rest() {
        return new JsonArrayWrapper(jsonArray, mark+1);
    }

    public int size() {
        return jsonArray.size() - mark;
    }

    public JsonElement get(int n) {
        return jsonArray.get(mark + n);
    }

    public void add(JsonElement jsonElement) {
        jsonArray.add(jsonElement);
    }

    public void addAll(JsonArrayWrapper jsonArrayWrapper) {
        jsonArrayWrapper.forEach(this.jsonArray::add);
    }

    @Override
    public Iterator<JsonElement> iterator() {
        JsonArray jsonarray = new JsonArray();
        this.forEach(e -> jsonarray.add(e));
        return jsonarray.iterator();
    }

    @Override
    public void forEach(Consumer<? super JsonElement> action) {
        for (int i=mark; i<jsonArray.size(); i++) {
            action.accept(jsonArray.get(i));
        }
    }
}
```

## 4 成果

经过这两个主要的优化，就解决了代码中的性能问题，成果如下图所示：

{% asset_img bg2016031202.png %}
