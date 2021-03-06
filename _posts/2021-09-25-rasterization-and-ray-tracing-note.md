---
layout: post
title:  "ray-tracing 笔记"
date:   2021-09-25 00:00:00 +0800
categories: cs cg
tag: ray-tracing
---

## 光线追踪

参考：什么是光线追踪，路径追踪，降噪？
> https://zhuanlan.zhihu.com/p/123866941

> 相比光追，光栅化最大的特点之一在于，它必须要有相机。在光栅化渲染中，要把相机放在待渲染的三角面片前，使用投影矩阵，经过变换以适应屏幕，将其渲染为2D画面上的各个像素。

> 而光线追踪可以执行渲染所需的计算，跟相机无关。这就是光线追踪具有高反射的原因。在光栅化过程中（当然也有其他的trick），我们只有在安装了额外的相机（如镜子）的情况下，才能正确地计算反射。而光线追踪可从任何地方发射光线，所以我们能用光追计算在任意一点上的反射。

视频介绍：
> https://zhuanlan.zhihu.com/p/165539118

## rust-ray-tracing

> https://github.com/peigongdh/rust-ray-tracing-demo

> https://raytracing.github.io/books/RayTracingInOneWeekend.html

> https://zhuanlan.zhihu.com/p/152300191

演示demo
> https://lucifier129.github.io/rust-ray-tracing-demo/ray-tracing-web/build/