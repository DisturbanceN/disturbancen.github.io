---
layout: post
title: 使用LayaBox解析xml文件
categories: [blog ]
tags: [游戏开发, ]
description:  
---

LayaBox对XML的加载进行了封装，相对于纯JS加载xmldom来说要方便了很多，我们直接调用laya的loader便可加载完成

```
var _res = [
	{url: "res/config/test.xml", type: laya.net.Loader.XML},
];

var loadXml = function(){
	//解析xml代码
}

Laya.loader.load(_res, laya.utils.Handler.create(this, function () {
            //加载完毕
            loadXml();
        }));
}
```

注意：加载文件的类型一定要是laya.net.Loader.XML

以下是用来测试的xml

```
<Root Name="test">
	<ATTR1 num="0" count="147"></ATTR1>
	<ATTR2 path="test/1024.jpg" name="你猜"></ATTR2>
	<REPEATED name="小明" age="10"></REPEATED>
	<REPEATED name="李狗蛋儿" age="20"></REPEATED>
</Root>
```

加载完成之后就是对xml文件的解析了，首先我们要获取这个xmldom

```
var xmlDom = laya.net.Loader.getRes("res/config/test.xml");
```

然后就可以逐层遍历xml，把数据按我们想要的格式存储起来

```
var attr = xmlDom.childNodes[0].childNodes;
for (var i = 0; i < attr.length; i++){
    if (attr[i].nodeName == "ATTR1"){
        for (var j = 0 ; j < attr[i].attributes.length ; j++){
            if (attr[i].attributes[j].nodeName == "num"){
                this._battleData.num = attr[i].attributes[j].nodeValue;
            }
            else if (attr[i].attributes[j].nodeName == "count"){
                this._battleData.count = attr[i].attributes[j].nodeValue;
            }
        }
    }
}
```

对于嵌套层次更深的节点，也是类似的做法
这里只是简单地记录一下使用方法，真正的游戏中需要把xml解析使用单独文件来管理