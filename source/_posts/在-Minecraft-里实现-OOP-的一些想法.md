---
title: 在 Minecraft 里实现 OOP 的一些想法
date: 2026-08-13 22:58:54
tags:
  - dovetail
  - minecraft
categories:
  - OOP
---

## 起因

Dovetail 的 OOP 一直是缺少实现，只有空话的。所以 最近在讨论 Dovetail 要不要引入 OOP 支持， 顺带聊了多态、 RTTI、虚函数表在
Minecraft 环境下怎么实现。 以下是一些不成熟的设想，以实际实现为准（

## 多态的两种初步方案

背景伪代码：

```python
class Base:
    def kill(self): pass


class Cat(Base):
    def kill(self): print("Killing cat")


class Dog(Base):
    def kill(self): print("Killing dog")


def kill(obj: Base):
    obj.kill()
``` 

### 方案一：编译器展开 isinstance 分支

obj.kill () 在编译时展开为：

```python
if type(obj) == Cat:
    Cat.kill(obj)
elif type(obj) == Dog:
    Dog.kill(obj)
```

- 优点：简单
- 缺点：子类数量一多，生成代码膨胀，性能堪忧

### 方案二：类型表 + 继承链表（基于函数路径）

Lua 风格伪代码：

```lua
Base = { foo = "base/foo" }
Cat  = { foo = "base/foo", kill = "cat/kill" }
Dog  = { foo = "dog/foo",  kill = "dog/kill" }

types = { [1]=Base, [2]=Cat, [3]=Dog }
super = { [2]={1}, [3]={1} }

-- obj.kill() 编译为：
-- types[obj.type].kill
```

每个对象携带整数类型 ID，调用时查 types 表拿到方法路径，再执行。 继承链通过 super 表记录，编译期通过C3算法线性化继承链，查找时沿着线性链查找。
同时无子类的类直接编译期内联，跳过查表

## Minecraft 环境的特殊性

- MC 没有指针，跳表通过函数路径 + function 宏模拟
- 跨编译单元的多态暂属 UB，OOP 类型设想不允许跨模块导出

## 未解决的问题/摆烂的部分

分离编译：两个文件各自继承 Base 分别编译，任意一方无法感知对方子类，所以这些操作通通按 UB 处理qwq

## 总结

MC 环境太特殊，标准 C++ 那套拿过来不能直接用。 最终方案大概是：类型表 + 继承链表 + 编译期内联优化混合的史山w。

## 鸣谢

感谢 4424 大佬提供的思路