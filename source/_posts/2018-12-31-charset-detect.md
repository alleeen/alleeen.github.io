---
layout: post
title:  字符编码的那些事
date: 2018-12-30 16:21:39
comments: true
ads: true
categories: [软件技术]
tags: [Java,字符编码]
---

最近在处理一个需求时发现业务字段出现了一串异常的字符。熟悉 web 开发的同学应该一眼就能看出，诸如`%C2%D6%BB%D8%CA%AF`之类的字符串是一个 URL Encoded 的字符串。导致这些未经 decode 的数据直接展示到界面上的原因是，业务日志中该字段格式不统一，有些是未经 URL Encoded 的，但有些又经过  URL Encoded 了，ETL 没有对这种情况进行处理就直接入库了。

<!-- more -->

为了解决这个问题，ETL 需要做的就是先要判断输入的字符串是否是  URL Encoded 的，如果不是，直接返回即可，如果不是则需要做进一步的处理

## 什么是 URL Encoding

URL encoding是Uniform Resource Identifier(URI)规范文档中对特殊字符编码制定的规则。本质是把一个字符转为百分号（%）加上其字符编码对应的16进制数字。故又称之为Percent-encoding。一般来说，URL只能使用英文字母、阿拉伯数字和某些标点符号，不能使用其他文字和符号。比如，世界上有英文字母的网址"http://www.abc.com"，但是没有希腊字母的网址"http://www.aβγ.com"（读作阿尔法-贝塔-伽玛.com）。所以如果 URL 中有中文，那么就必须进行 URL Encoding 。但是麻烦的是，相关规范并没有规定具体的编码方法，而是交给应用程序（浏览器）自己决定，比如对于中文字符，应用程序可以先使用 UFT-8 编码后，再 URL encoding，也可以使用 GBK 编码后，再  URL encoding，这两种实现方式都是合法的，但最终产生的结果并不一样。这导致"URL编码"成为了一个混乱的领域(详细可以参考：[关于URL编码](http://www.ruanyifeng.com/blog/2010/02/url_encoding.html))。

## 字符编码识别

我们从数据中抽取了一些 URL encoded 数据并使用一些在线解析工具进行了解析，发现它们的原始编码是 GBK 的，由于担心数据中混有其他的编码格式，我们想如何自动的识别字符的编码格式。

### 使用`juniversalchardet`做字符编码识别

经过一番搜索，找到了一个叫`juniversalchardet`的工具。 它是Mozilla 公司的 firefox 使用的 universalchardet 编码自动检测工具的 Java 版本。自动编码主要是根据统计学的方法来判断。具体原理，可以看：[A composite approach to language/encoding detection](https://www-archive.mozilla.org/projects/intl/UniversalCharsetDetection.html)

下面写个小例子来验证他的特性，首先使用 maven 引入依赖

```xml
<!-- Mozilla的编码识别包 -->
<dependency>
    <groupId>com.googlecode.juniversalchardet</groupId>
    <artifactId>juniversalchardet</artifactId>
    <version>1.0.3</version>
</dependency>
```

写个简单的Demo

```java
import java.io.File;
import java.io.IOException;
import looly.github.hutool.FileUtil;
import org.mozilla.universalchardet.UniversalDetector;
/**
 * 编码识别工具类
 * @author allanzheng
 *
 */
public class CharsetDetectUtil {
    public static String detect(byte[] content) {
        UniversalDetector detector = new UniversalDetector(null);
        //开始给一部分数据，让学习一下啊，官方建议是1000个byte左右（当然这1000个byte你得包含中文之类的）
        detector.handleData(content, 0, content.length);
        //识别结束必须调用这个方法
        detector.dataEnd();
        return detector.getDetectedCharset();
    }
    public static void main(String[] args) throws IOException {
        byte[] bytes = FileUtil.readBytes(new File("E:/workspace/python/htmlUtil.txt"));
        System.out.println(detect(bytes));
    }
}
```

经过实际验证，发现该类库在识别长文本的时候准确率还是挺高的（如 1000 个byte左右），但是短文本就，比如我们的道具名称字段，就无能为力了。类似的编码识别库还有 [ICU](http://site.icu-project.org/)。

### 使用`java.nio.charset.CharsetDecoder`自动识别字符集

一般用两种方法构建InputStreamReader：

```java
InputStreamReader reader = new InputStreamReader(in, charsetName);

or

InputStreamReader reader = new InputStreamReader(in, charset);
```

如果charset不匹配，则输出乱码。还有一种构建方法，即利用CharsetDecoder：

```java
CharsetDecoder cd = charset.newDecoder();
InputStreamReader reader = new InputStreamReader(in, cd);
```

这时如果不匹配，则抛出异常：

```java
java.nio.charset.MalformedInputException: Input length = 1
    at java.nio.charset.CoderResult.throwException(CoderResult.java:277)
    at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:338)
    at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:177)
        ....
```

这样，就可以用作字符集探测。所以我们的解决方案如下，AutoCharsetReader 用于探测字符集

```java
  public class AutoCharsetReader {
    private final static String[] _defaultCharsets = {
        "US-ASCII",
        "UTF-8",
        "GBK",
        "GB2312",
        "BIG5",
        "GB18030",
        "UTF-16BE",
        "UTF-16LE",
        "UTF-16",
        "UNICODE"};



    public static Charset detectCharset(byte[] bytes, String[] charsets) {

        Charset charset = null;

        for (String charsetName : charsets) {
            charset = detectCharset(bytes, Charset.forName(charsetName));
            if (charset != null) {
                break;
            }
        }

        return charset;
    }

    private static Charset detectCharset(byte[] bytes, Charset charset) {
        try {
            BufferedInputStream input = new BufferedInputStream(new ByteArrayInputStream(bytes));

            CharsetDecoder decoder = charset.newDecoder();
            decoder.reset();

            byte[] buffer = new byte[512];
            boolean identified = false;
            while ((input.read(buffer) != -1) && (!identified)) {
                identified = identify(buffer, decoder);
            }

            input.close();

            if (identified) {
                return charset;
            } else {
                return null;
            }

        } catch (Exception e) {
            return null;
        }
    }

    private static boolean identify(byte[] bytes, CharsetDecoder decoder) {
        try {
            decoder.decode(ByteBuffer.wrap(bytes));
        } catch (CharacterCodingException e) {
            return false;
        }
        return true;
    }
}
```

```scala
private[ams] def decodeItemName(str: String): String = {
    // 由于item name
    val decodedStr = URLDecoder.decode(str, "UTF-8")
    if (decodedStr.equals(str)) {
      str
    } else {
      val bytes = URLCodec.decodeUrl(str.getBytes(StandardCharsets.US_ASCII))
      val charSet = AutoCharsetReader.detectCharset(bytes, Array[String]("UTF-8", "GBK"))
      new String(bytes, charSet)
    }
  }
```
