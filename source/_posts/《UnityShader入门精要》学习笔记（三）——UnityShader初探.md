---
title: 《UnityShader入门精要》学习笔记（三）——UnityShader初探
date: 2020-05-26 22:33:35
categories: UnityShader学习笔记
tags: 
- Unity
- Shader
- 学习笔记
---
---
## 何为Unity Shader
在传统的开发模式中，开发者如果想要设置自定义的渲染模式，需要和大量文件和设置打交道（包括编写顶点着色器、片元着色器、选择图形API、加载资源到GPU等等），非常繁琐复杂。Unity Shader就是在此之上的更高级的一层封装，开发者只需要在Shader Lab中编写Unity Shader文件，即可交由Unity引擎实现自定义渲染效果。
因此Unity Shader与传统的Shader有所不同，它定义的是要显示一个材质的所有东西，**而不仅仅是着色器代码**。

而在使用Unity Shader时，我们首先需要创建一个材质，将编写的Unity Shader文件挂载到该材质上，之后再为各个物体应用该材质。由此可以看出，Unity Shader是与材质牢牢绑定的。

## Unity Shader文件基本结构
一个UnityShader的基本文件结构大致如下
```
Shader "ShaderName"{
  Properties{
    //相关属性
  }
  SubShader{
  //显卡A使用的子着色器
  }
  SubShader{
  //显卡B使用的子着色器
  }
  Fallback "VertexLit"
}
```
一个完整的Unity Shader包括shader名 ShaderName、shader包含的属性Properties、使用的子着色器SubShader、以及无法调用任何子着色器时执行的Fallback

#### Properties
Properties定义了一系列属性，这些属性将会出现在材质面板中，Properties语句块的定义如下：
```
Properties{
  Name("display name",PropertyType) = DefaultValue
  Name("display name",PropertyType) = DefaultValue
}
```
- Name为属性名，可以在后续代码中使用，**一般以下划线开头**
- display name指的是在材质面板中显示的名称
- PropertyType为属性类型
- DefaultValue为属性的默认值，第一次为某个材质应用该Shader时就会使用该默认值

在properties声明变量后，我们还需要在cg代码中定义变量来进行使用。
需要注意的是，即使我们不在properties中声明变量，我们也可以通过脚本的方式向shader传递变量的值，因此properties块的功能仅仅是为了让对应属性出现在材质面板中。

#### SubShader
SubShader主要包含这样几个部分
```
SubShader{
  [Tags]
  [RenderSetup]
  Pass{
    ...
  }
}
```
- 其中 标签是一系列“键值对”，用来进行和渲染引擎的“沟通”，告知其应该怎样、何时渲染这个对象
- 渲染设置是用来设置显卡渲染时的一些选项，如是否开启混合等
- Pass是最重要的语句块，可以有多个，每个Pass在渲染流程中执行一次，但是为了避免多个pass造成性能下降，我们应该尽可能用最少的Pass实现渲染功能。

对于每个Pass，结构类似SubShader
```
Pass{
  [name]
  [Tags]
  [RenderSetup]
    ...
}
```
- 名称：可以设置Pass的名称，这样我们在其他SubShader中，可以通过UsePass来使用其他Shader中定义的Pass，实现代码复用（需要指明路径）。注意，Unity会自动将Pass的名称转换为全大写字母
- 标签：其中Pass可以使用的标签和SubShader略有不同
- 渲染设置：和SubShader一致。如果我们在SubShader中进行设置，则会应用于所有Pass。如果不想这样，可以分别为每个Pass设置不同的RenderSetup

一个UnityShader文件中可以包含多个SubShader，主要用来实现不同显卡上的兼容性，如果第一个SubShader中的某些指令显卡不支持，则会使用下面的SubShader，以此类推。如果没有一个SubShader兼容，则会使用Fallback

#### Fallback
Fallback可以认为是所有SubShader都无法使用时最迫不得已的选项，是最基础的渲染。当然也可以手动关闭，意味着如果没有SubShader支持，我们就不去管他。
