# Rakudo and NQP Internals Workshop

## What's in here?

This repository contains course material for a workshop on Rakudo and NQP
internals. In here you'll find:

* The original source (build instructions below)
* [Pre-built PDFs](http://edumentab.github.io/rakudo-and-nqp-internals-course/)

## Abstract

This intensive 2-day workshop takes a deep dive into many areas of the Rakudo
Perl 6 and NQP internals, mostly focusing on the backend-agnostic parts but
with some coverage of the JVM and future MoarVM backends also. During the
course, participants will build their own small compiler, complete with a
simple class-based object system, to help them understand how the toolchain
works.

### Prerequisites

A reasonable knowledge of the Perl 6 language and, preferably, a little
experience working with NQP also.

### 第一天

##### 鹰的视角: 编译器和 NQP/Rakudo 架构

* 编译器做什么
* 运行时做什么
* Perl 6 的挑战
* NQP 作为语言
* NQP 作为编译器构建工具链
* QAST
* nqp:: op 集合
* Bootstrapping in a nutshell
* Rakudo 如何使用 NQP

#### NQP 语言

* 设计目标
* 字面值, 变量, 控制流
* 子例程, pointy blocks, 闭包语义
* 类, 属性, 方法
* 正则表达式和grammars
* Roles
* 多重分派
* 内置和 nqp:: ops
* 异常处理
* 限制和与完整 Perl 6 的其它差异
* 缺点

#### 编译管道

* HLL::Compiler 类
* 前端, 后端和它们之间的 QAST
* 用 grammars 解析, 用 actions 构建 AST
* 代码生成
* 从头开始构建一个微小的语言

#### QAST

* QAST::Node, 这一切的基础
* AST 的总体结构
* 字面值: QAST::IVal, QAST::NVal 和 QAST::SVal
* 运算: QAST::Op, 基本的例子
* 序列: QAST::Stmts, QAST::Stmt
* 变量: QAST::Var, QAST::VarWithFallback, scope, decl, value
* block 符号表
* 调用
* 形参和实参
* 上下文化: QAST::Want
* Block 引用: QAST::BVal
* 对象引用: QAST::WVal
* The backend escape hatch: QAST::VM
* At the top: QAST::CompUnit

#### 探索 nqp:: ops

* 计算
* 关系
* 聚集
* 字符串
* 流程控制
* 异常相关
* 上下文自省
* 大整数

### 第二天

#### 6model

* 对象: 行为 + 状态
* 类型和种类
* 元对象
* Representations
* STables
* 知其所以然, 这一切的根本
* 从头开始构建一个简单的对象系统
* Adding objects to our little language, using a World
* 方法缓存
* 类型检查
* Boolification
* Invocation
* 探索 NQP 的元对象
* 探索 Rakudo 的元对象
* 容器处理

#### 有界序列化和模块加载

* 编译时/运行时边界
* 序列化上下文
* 重访 QAST::WVal
* 什么是"有界的"
* Repossession, 冲突和其它令人讨厌的东西
* 与序列化上下文相关的 nqp:: ops
* The World, revisisted
* 模块加载的工作原理

#### 正则表达式和 grammar 引擎

* QAST::Regex 节点和它的子类型
* Cursor 和 Match
* The bstack and the cstack
* NFAs 和 Longest Token Matching

#### JVM 后端

* JVM 概述
* QAST 到 JAST 的翻译
* 运行时支持库

#### MoarVM 后端

* MoarVM 概述
* QAST 到 MAST 的翻译

## Build Instructions

Run the `Makefile` to build the slides for the two days. To do this, you will
need:

* A `make` program (`nmake` on Windows works just fine too)
* Perl 5.10 or above
* Pandoc (see http://johnmacfarlane.net/pandoc/)
* The `latex`, `dvips` and `ps2pdf` commands in your path

On Linux you can do something like:

    apt-get install texlive
    apt-get install pandoc

On Windows, there is a Pandoc installer from the URL mentioned above, then
install MiKTeX for the other commands.

## Course Delivery

This course material is made available by Edument AB under a Creative Commons
license (see LICENSE file) to support the Perl 6 development community. It is,
however, best experienced live! If you're interested in having this material
delivered by an experienced instructor at a location of your choice, feel free
to contact us at info@edument.se. To learn more about Edument and our other
awesome courses, see http://edument.se/.
