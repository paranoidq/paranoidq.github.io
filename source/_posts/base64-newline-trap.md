title: BASE64换行符的坑
date: 2016-08-01 21:29:38
tags: [base64, java]
categories: java
---

### 问题描述
client发报文给server，server端接收到http post发来的报文之后做base64解码，然后通过消息中间件传给核心系统进行处理。然后发现，client端发过来的报文会多出很多`/r/n`这样的字符，导致核心系统base64解码后也多出了`/r/n`，而核心系统之前没有考虑到这些问题，从而报错。

<!--more-->

### 问题的分析
问题在于为什么发送方的报文会多出来`/r/n`呢？
#### Step 1
首先看回车和换行符的区别：
>在计算机还没有出现之前，有一种叫做电传打字机（Teletype Model 33）的玩意，每秒钟可以打10个字符。但是它有一个问题，就是打完一行换行的时候，要用去0.2秒，正好可以打两个字符。要是在这0.2秒里面，又有新的字符传过来，那么这个字符将丢失。
     于是，研制人员想了个办法解决这个问题，就是在每行后面加两个表示结束的字符。一个叫做“回车”，告诉打字机把打印头定位在左边界；另一个叫做“换行”，告诉打字机把纸向下移一行。
这就是“换行”和“回车”的来历，从它们的英语名字上也可以看出一二。
      后来，计算机发明了，这两个概念也就被般到了计算机上。那时，存储器很贵，一些科学家认为在每行结尾加两个字符太浪费了，加一个就可以。于是，就出现了分歧。
Unix 系统里，每行结尾只有“<换行>”，即“\n”；Windows系统里面，每行结尾是“ <回车><换 行>”，即“\r\n”；Mac系统里，每行结尾是“<回车>”。一个直接后果是，Unix/Mac系统下的文件在Windows里打 开的话，所有文字会变成一行；而Windows里的文件在Unix/Mac下打开的话，在每行的结尾可能会多出一个^M符号。

所以导致的问题应该就是client端是windows系统，而我们这边处理的系统在linux下，因此就会有`/r/n`的问题。

#### Step 2
OK，让对方去掉报文中的换行之后，问题还是存在。而且还有新的发现：
>BASE64之后，当字符串过长（一般超过76）时会自动在中间加一个换行符。及时我们自己测试的报文完全没有任何换行存在。

于是想办法去研究`sun.misc.BASE64Encoder`的源码，有了一些发现。

BASE64主要调用的方法是：
```java
byte[] bytes = "abcd".getBytes();
BASE64Encoder encoder = new BASE64Encoder();
encoder.encodeBuffer(bytes);
```

encodeBuffer源码的大致情况：(大部分源码位于`BASE64Encoder`的父类`CharacterEncoder`中)
```
public String encodeBuffer(byte aBuffer[]) {
    ByteArrayOutputStream   outStream = new ByteArrayOutputStream();
    ByteArrayInputStream    inStream = new ByteArrayInputStream(aBuffer);
    try {
        encodeBuffer(inStream, outStream);
    } catch (Exception IOException) {
        // This should never happen.
        throw new Error("CharacterEncoder.encodeBuffer internal error");
    }
    return (outStream.toString());
}

/**
 * Encode bytes from the input stream, and write them as text characters
 * to the output stream. This method will run until it exhausts the
 * input stream, but does not print the line suffix for a final
 * line that is shorter than bytesPerLine().
 */
public void encode(InputStream inStream, OutputStream outStream)
    throws IOException {
    int     j;
    int     numBytes;
    byte    tmpbuffer[] = new byte[bytesPerLine()];

    encodeBufferPrefix(outStream); 

    while (true) {
        numBytes = readFully(inStream, tmpbuffer);
        if (numBytes == 0) {
            break;
        }
        encodeLinePrefix(outStream, numBytes); 
        for (j = 0; j < numBytes; j += bytesPerAtom()) {

            if ((j + bytesPerAtom()) <= numBytes) {
                encodeAtom(outStream, tmpbuffer, j, bytesPerAtom());
            } else {
                encodeAtom(outStream, tmpbuffer, j, (numBytes)- j);
            }
        }
        if (numBytes < bytesPerLine()) {
            break;
        } else {
            encodeLineSuffix(outStream); // 这一行会输出换行
        }
    }
    encodeBufferSuffix(outStream); 
}

/**
 * Encode the suffix that ends every output line. By default
 * this method just prints a <newline> into the output stream.
 */
protected void encodeLineSuffix(OutputStream aStream) throws IOException {
    pStream.println();
}
```

注意，`encodeLineSuffix`会输出换行。也就是每次读满一个buffer大小的时候，都会输出一个换行。buffer的大小是由`bytesPerLine()`函数决定的，该函数是一个抽象函数，由子类实现。而在BASE64Encoder中，该函数的返回值为57.
```
/**
 * this class encodes 57 bytes per line. This results in a maximum
 * of 57/3 * 4 or 76 characters per output line. Not counting the
 * line termination.
 */
protected int bytesPerLine() {
    return (57);
}
```
[StackOverflow上有回答](http://stackoverflow.com/questions/9341047/carriage-return-issue-decoding-base64-from-java-and-sending-to-browser)这是做了一种`chunking`，在每一个`chunk`后面添加了`/n`。并且sun的库函数只存在于oracle的jvm下面，而不存在于其他jvm中。

### Step 3
`encode`与`encodeBuffer`有细微的区别：`encodeBuffer`会在最后一行不足`bytesPerline()`时添加一个换行符，而encode则不会做处理。

貌似很坑爹，做了如此多的隐含处理，让调用者想死的心都有了。


### 一劳永逸的办法

建议使用`org.apache.commons.codec.binary.Base64`库：
```
Base64.encodeBase64(..);
Base64.decodeBase64(..)
```

并且该库显示指明了，你是否需要`chunk`选项和`urlsafe`选项（避免输出`+`和`/`，而是输出`-`和`_`）：
```
encodeBase64Chunked(final byte[] binaryData)
encodeBase64(final byte[] binaryData, final boolean isChunked)
```






