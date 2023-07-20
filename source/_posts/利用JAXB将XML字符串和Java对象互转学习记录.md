---
title: 利用JAXB将XML字符串和Java对象互转学习记录
date: 2023-07-20 10:44:04
tags: [Java,xml,JAXB]
categories: Java
---

# 代码实现

话不多说，直接上工具类代码：

```
public class XmlUtil {

    /**
     * @param xml xml字符串
     * @param obj     要转换的实体类型
     * @return : java.util.Map<java.lang.String,java.lang.String>
     * @Name : xmlToObject
     * @description : xml字符串内容并转化为对应实体
     * @createTime : 2023/7/14 14:04
     */
    static public <T> T xmlToObject(String xml, Class<T> obj) throws JAXBException {
        StringReader stringReader = new StringReader(xml);
        JAXBContext jaxbContext = JAXBContext.newInstance(obj);
        Unmarshaller unmarshaller = jaxbContext.createUnmarshaller();
        Object unmarshal = unmarshaller.unmarshal(stringReader);
        return obj.cast(unmarshal);
    }

    /**
     * @Name : objToXml
     * @description : 将Java对象转换成为XML文本
     * @createTime : 2023/7/20 10:47
     * @param obj  要转换的实体类型
     * @return : java.lang.String
     */
    static public String objToXml(Object obj) throws JAXBException, IOException {
        JAXBContext jaxbContext = JAXBContext.newInstance(obj.getClass());
        Marshaller marshaller = jaxbContext.createMarshaller();
        StringWriter sw = new StringWriter();
        marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, Boolean.TRUE);
        marshaller.marshal(obj,sw);
        String res = sw.toString();
        sw.close();
        // 尖括号修正
        res = res.replace("&lt;","<");
        res = res.replace("&gt;",">");
        return res;
    }
    
}

```

再上实体类代码：

```
@Data
@XmlAccessorType(XmlAccessType.FIELD)
@XmlRootElement(name = "xml")
public abstract class WXMessage implements Serializable {

    @XmlElement(name = "ToUserName",required = true)
    private String toUserName;

    @XmlElement(name = "FromUserName",required = true)
    private String fromUserName;

    @XmlElement(name = "MsgType",required = true)
    private String msgType;

    @XmlElement(name = "CreateTime")
    private String createTime;

    @XmlElement(name = "MsgId")
    private String msgId;

    @XmlElement(name = "MsgDataId")
    private String msgDataId;

    @XmlElement(name = "Idx")
    private String idx;

}
```

值得注意的是JAXB的那几个注解：

- `@XmlRootElement(name = "xml")` 映射的根节点标签名，未使用则默认根节点是该实体的全类名。
- `@XmlAccessorType(XmlAccessType.FIELD)` 映射属性形式，可映射字段(FIELD)、Setter/Getter方法(PROPERTY)等。
-  `@XmlElement` 属性值，映射着XML的具体标签名。

上面三个注解算是最基本的注解了。现在如过有个需求，需要对转换的XML字段进行具体的微调操作，比如我要给一些字段变成`<![CDATA[原字段]]>`的形式，那么怎么实现？

*答曰，配合`@XmlJavaTypeAdapter`注解实现自定义转换器。*

转换器实现：

```
public class XmlCDataAdapter extends XmlAdapter<String,String> {

    @Override
    public String unmarshal(String v) throws Exception {
        return v;
    }

    @Override
    public String marshal(String v) throws Exception {
        return "<![CDATA[" + v + "]]>";
    }
}
```

其实也就是重写两个转义的方法，对具体数据就可以实现微调了。

实体代码示例：

```
@Data
@XmlAccessorType(XmlAccessType.FIELD)
@XmlRootElement(name = "xml")
public abstract class WXMessage implements Serializable {

    @XmlElement(name = "ToUserName",required = true)
    @XmlJavaTypeAdapter(XmlCDataAdapter.class)	// 对应要微调的字段添加该注解就行了
    private String toUserName;

}
```

想法很不错，实际用起来存在一个很大的问题：**尖括号<>在解析的时候会被JAXB自动转义成`&lt/&gt`!**

这下咋整？博主当时研究了半天，网上的方法都是什么更换转码器啊，还要用代理的，怎么一个这样的小问题搞这么复杂啊。

其实你转义过去处理结果字符串的时候再**替换**回来不就行了？

```
    /**
     * @Name : objToXml
     * @description : 将Java对象转换成为XML文本
     * @createTime : 2023/7/20 10:47
     * @param obj  要转换的实体类型
     * @return : java.lang.String
     */
    static public String objToXml(Object obj) throws JAXBException, IOException {
        JAXBContext jaxbContext = JAXBContext.newInstance(obj.getClass());
        Marshaller marshaller = jaxbContext.createMarshaller();
        StringWriter sw = new StringWriter();
        marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, Boolean.TRUE);
        marshaller.marshal(obj,sw);
        String res = sw.toString();
        sw.close();
        // 尖括号修正
        res = res.replace("&lt;","<");
        res = res.replace("&gt;",">");
        return res;
    }
```

工具类里给结果的尖括号修正一下就好了。。。

## 深层次问题

代码实现简单，就是用了JAXB的转换工具。博主遇到的问题主要是在XML映射的对象上。

现在我有一个父类，若干继承了该父类的子类，父类的字段是子类共有的。

父类：

```
@Data
@XmlAccessorType(XmlAccessType.FIELD)
@XmlRootElement(name = "xml")
public class WXMessage implements Serializable {

    @XmlElement(name = "ToUserName",required = true)
    private String toUserName;

    @XmlElement(name = "FromUserName",required = true)
    private String fromUserName;

    @XmlElement(name = "MsgType",required = true)
    private String msgType;

    @XmlElement(name = "CreateTime")
    private String createTime;

    @XmlElement(name = "MsgId")
    private String msgId;

    @XmlElement(name = "MsgDataId")
    private String msgDataId;

    @XmlElement(name = "Idx")
    private String idx;

}
```

子类：

```
@Data
@EqualsAndHashCode(callSuper = true)
@XmlAccessorType(XmlAccessType.FIELD)
public class WXTextMessage extends WXMessage {

    @XmlElement(name = "Content")
    private String content;

    public WXTextMessage() {
        super();
        this.setMsgType(WXMessageType.TEXT.getType());
    }
}
```

使用我自己写的工具类的时候报错：

![image-20230720112053365](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230720112053365.png)

明显是根节点无法读取，本来我以为父类声明了`@XmlRootElement(name = "xml")`，他的子类应该都带有这个注解的效果了，结果就报错了。

那么子类加上这个注解吧。

```
@Data
@EqualsAndHashCode(callSuper = true)
@XmlAccessorType(XmlAccessType.FIELD)
@XmlRootElement(name = "xml")	// 加上根节点标记
public class WXTextMessage extends WXMessage {

    @XmlElement(name = "Content")
    private String content;

    public WXTextMessage() {
        super();
        this.setMsgType(WXMessageType.TEXT.getType());
    }
}
```

结果发现出现了新的报错：

![image-20230720112933842](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230720112933842.png)

类型转换错误？还是父类无法往子类转换。这怎么就变父类了呢，我都是直接把子类的`Class`传进去的啊，打上断点调试一波看看：

![image-20230720115807447](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230720115807447.png)

这下更加糊涂了，可见传入的`obj`对象是子类类型，调用`unmarshaller`方法后转出来的竟然是它的父类！

博主思前想后，不懂其中的奥义，秉着~~排列组合~~的思想，既然父子类都标记，只有父类标记都用了，那就试试**只有子类标记**吧。好家伙，试一下就直接给我解决了。

![image-20230720120540937](https://fastly.jsdelivr.net/gh/Echo-xzp/Resource/img/image-20230720120540937.png)

好好好，一下就给我解决了。

那么试着分析一波，JAXB解析的时候，读取`@XmlRootElement(name = "xml")`标签，当有父类有此标记的时候，**优先读取父类的标签**，并将其转换为父类对象。当然这也是我自己的分析，实际上最好对着源码分析一波，但是博主是在太懒了，有兴趣的研究看看。

其实到底这种手动解析XML的形式实在有点啰嗦，还有很多方法直接在Spring里面直接像解析`Json`一样，请求过来和返回都直接交给Spring的转换器来操作，这样才是最佳方案。
