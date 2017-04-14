---
layout: post
title: NGUI源码分析之----UIDrawCall
categories: [blog ]
tags: [NGUI,游戏开发 ]
description:  
---

## 基础知识
### 1.在Unity中如何确定渲染的顺序
相关因素: Render Queue、 ZWrite、ZTest  
1.Unity会先渲染Render Queue中靠前的物体  
2.ZWrite取值为Off，这时候只要是后渲染的就会覆盖前面渲染的相同位置的像素  
3.ZWrite取值为On，这时候后渲染的物体会和深度缓冲区中的深度进行比较，如果比较成功了，则覆盖之前的像素。否则继续使用原来的像素  
4.深度的比较规则由ZTest设置，一共有以下参数类型：

|---
|类型名 |描述
|:-|:-
|Greater |只渲染大于AlphaValue值的像素
|GEqual |只渲染大于等于AlphaValue值的像素
|Less |只渲染小于AlphaValue值的像素
|LEqual |只渲染小于等于AlphaValue值的像素
|Equal |只渲染等于AlphaValue值的像素
|NotEqual |只渲染不等于AlphaValue值的像素
|Always |渲染所有像素，等于关闭透明度测试。等于用AlphaTest Off
|Never |不渲染任何像素

## UIDrawCall
这是一个相对比较独立的类，抛开NGUI相关的内容，即使是自己实现一套代码，把渲染需要的关键数据传输进来也是可以使用的。每一个UIDrawCall对应一次draw call（一次GPU绘制）。它创建出一个GameObject并设置MeshFilter、Mesh、MeshRenderer、Material的信息，剩下的就交给Unity了。  
### 关键变量：
### mActiveList
处于激活状态的UIDrawCall的列表  
### mInactiveList
处于未激活状态的UIDrawCall的列表，相当于一个内存池，需要创建的时候优先从这个列表中拿  
### 关键函数：
### static UIDrawCall Create (string name)
创建一个UIDrawCall，如果mInactiveList中有元素则直接从列表中拿出来一个，加入mActiveList  
否则创建一个新的UIDrawCall  
```
GameObject go = new GameObject(name);
DontDestroyOnLoad(go);   //切换场景时不会释放
UIDrawCall newDC = go.AddComponent<UIDrawCall>();
mActiveList.Add(newDC);
```
### public void UpdateGeometry ()
外部的函数设置UIDrawCall的以下参数之后：

|---
|变量名 |描述
|:-|:-
|depthStart |开始深度
|depthEnd |结束深度
|panel |这个draw call对应的Panel
|verts |顶点
|uvs |贴图坐标（和顶点对应）
|cols |颜色（和顶点对应）
|norms |法线（和顶点对应）
|tans |切线（和顶点对应）

调用UpdateGeometry更新UIDrawCall对应的GameObject的MeshFilter、Mesh、MeshRenderer、Material的信息  
1.vertices、uv、colors、normals、tangents全部存储在Mesh中  
2.triangles数组表示生成的三角网格，长度一定为3的倍数，每3个点代表一个三角形网格。NGUI中的网格排列规则是固定的，使用GenerateCachedIndexBuffer函数根据顶点数量生成。4个点为一组，包含两个三角形网格，具体规则详见GenerateCachedIndexBuffer  
3.对于Mesh赋值的官方建议：  
- 建议先赋值顶点数组之后再赋值三角形数组，为了避免越界的错误
- 调用Clean函数在赋予新的顶点值和三角形索引值之前是非常重要的，Unity总是检查三角形的索引值，判断它们是否超出边界。调用Clear函数后，给顶点赋值，再给三角形数组赋值，以确保没有超出数组的边界

4.后续这个gameobject就会根据Unity自身的渲染规则进行渲染了  
5.渲染顺序根据renderQueue的值确定。这个值实际上是外部程序来设置的（详细原理见基础知识【1】）  

### int[] GenerateCachedIndexBuffer (int vertexCount, int indexCount)
这个函数很简单，传入顶点的数量，求出网格对应顶点的下标。由于NGUI的网格是是有规律的，所以生成规则是一致的。假设有4个顶点1,2,3,4。那么1、2、3组成一个网格。3、4、1组成一个网格。每四个顶点一组，对应2个三角形网格  

### void OnWillRenderObject ()
MonoBehaviour的函数，当渲染物体之前，如果对象可见每个相机都会调用它  
1.调用UpdateMaterials检测是否需要重建纹理  
2.计算裁切信息  
他会从自身开始，寻找是否有裁切框，根据Unity编辑面板中的父子关系一层层网上找，如果遇到有裁切框，就计算出裁切框坐标、裁切的soft（边缘渐变）程度、相对旋转角度（如果是在自己的panel裁切，肯定是0）  
最后调用SetClipping把裁切信息写入纹理当中。最多只能嵌套3层裁切，分别对应ClipRange和ClipArgs中的3个字符串。（实际上一共有4个字符串，不知道第4个是干嘛的，对应的shader最多只有3层裁切）  
这几个串会在对应的shader中被用来实现真正的裁切  
PS：NGUI的Shader一般分为XXX、XXX 1、XXX 2、XXX 3，分别代表没有裁切、1次裁切、2次裁切和3次裁切  


### void UpdateMaterials ()
该函数在两种情况下会调用  
1.UpdateGeometry被调用时  
2.每次要渲染前OnWillRenderObject被调用时  
它负责检查是否需要重建纹理（不存在纹理或者被裁切的次数变了）  
如果需要重建，则调用RebuildMaterial重建纹理  

### Material RebuildMaterial ()
实际上就是销毁之前创建的Material，然后调用CreateMaterial再创建一个  
然后设置mRenderer的材质为新创建的材质  

### void CreateMaterial ()
创建纹理的时候首先会根据裁切次数寻找正确的shader。使用这个shader和baseMaterial（用来记录这个draw call的纹理），创建一个Material  

参考资料：
【1】渲染次序：http://www.tuicool.com/articles/VjiE3aQ  