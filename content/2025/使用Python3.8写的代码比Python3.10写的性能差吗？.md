---
title: "使用Python3.8写的代码比Python3.10写的代码性能差吗？"
date: 2025-01-22T11:21:26+08:00

categories:
  - 编程语言
  - Python
tags:
  - Python3.8
  - Python3.10

summary: "Python3.10 性能优于 3.8，但代码性能主要取决于运行环境版本，而非编写版本。开源项目选择 Python3.8 作为最低支持版本无需担心性能问题。"

---

一般情况下，Python3.10 的性能是要好于 Python3.8 的。那么是否意味着同等条件下，使用 Python3.8 写出来的代码要比 Python3.10 写出来的代码性能差呢？

笔者曾经写过一个项目，项目一开始使用 Python3.8。重构时，因为 3.8 不支持某些功能，一度将 Python 版本升到了 Python3.10。升到 3.10 后，除了能使用新功能之外，一个意外的发现是性能提升了 30% 左右！之前知道 Python 3.10 比 3.8 版本性能要好，但是不知道某些情况下性能居然能相差 30%。

如果只是一般项目，那选择 Python3.10 无疑是明智的，然而笔者的这个项目是开源的，那么就需要权衡一下版本的问题了。

目前 Python 主流的第三方库，一般最低支持版本是 Python3.8。从这些开源项目来看，显然对于目前来说 Python3.8 是一个经过各方考虑后的一个较为妥当的最低支持版本。

作为一个开源项目，笔者的这个项目也不能特立独行，还是跟随主流吧。于是笔者在项目重构完成后，尝试着将最低支持版本降回 Python3.8。对于需要用到的 Python3.10 的新功能，改为其他方式实现，最后成功地将版本降回了 3.8。

虽然成功降回 3.8，但是看着那降低了 30% 的性能，感觉挺不舒服的。

其实当时笔者陷入了一个误区，即觉得性能较低的 Python 版本写出来的代码性能也较低。后来回过头来，觉得代码的性能跟 Python 的版本没有必然联系，跟运行代码时使用的 Python 版本有很大的关系。

笔者使用 Python3.10 的环境重新运行了一下用 Python3.8 写的项目，果然性能和之前使用 Python3.10 写的性能没什么区别。

综上，就目前来说，开源项目选择 Python3.8 是无需顾虑性能问题的。