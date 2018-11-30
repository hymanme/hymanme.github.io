---
layout: post
title: Android Studio 常用快捷键
summary: 
date: 2018-11-29 22:35:33
categories: IDE
tags: [Android, Android Studio, Key Map]
featured-img: beach
---

> Mac OS 10.5+

### IDE 

| 按键                   | 说明                               |
| ---------------------- | ---------------------------------- |
| Option + 1             | *快速打开或隐藏工程面板            |
| Command +              | 打开 Project Structure 窗口        |
| Command + Option + Y   | 同步(sync with file system)        |
| Option + Shift + C     | 查看文件的变更记录                 |
| Control + R            | *运行                              |
| Control + D            | Debug运行                          |
| Command + Option + F12 | 资源管理器打开文件夹               |
| ESC                    | 光标返回到当前编辑框内             |
| Shift + Esc            | 光标返回到编辑框并且关闭无用窗口   |
| F12                    | 光标从编辑框返回最近使用的工具窗口 |

### 编辑

| 按键                        | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ |
| Command + C                 | 复制当前行或选中内容                                         |
| Command + D                 | 粘贴当前行或选中内容（不复制）                               |
| Command + X                 | 剪切当前行或选中的内容                                       |
| Command + Y                 | 以小窗模式展示当前光标选中的方法或类的定义                   |
| Command + Z                 | 撤销                                                         |
| Command + Shift + Z         | 反撤销                                                       |
| Option + Enter              | 自动修正                                                     |
| Command + Enter             | 在当前行下方添加空白行，光标不动                             |
| Shift + Enter               | 在当前行下方添加空白行，且光标移动到新行                     |
| Option + Command + L        | *格式化代码                                                  |
| Control + Option +O         | *优化导入的类和包                                            |
| Control + I                 | *实现方法                                                    |
| Control + O                 | *重写或实现方法                                              |
| Command + B                 | *跳到实现类或方法的定义处                                    |
| Command + U                 | *跳到父类或方法的定义处                                      |
| Command + [                 | *跳到上一个光标停留处                                        |
| Command + ]                 | *跳到下一个光标停留处                                        |
| Command + /                 | *注释或反注释当前行                                          |
| Command + Shift + /         | 块状注释或反注释当前选中代码                                 |
| Command + J                 | 选择代码模版并插入至当前光标处                               |
| Command + Option + T        | 把选中的代码放在 `try{}` 、`if{}` 、 `else{}` <br />等代码块中 |
| F2/ Shift F2                | 跳到下一个或上一个错误处                                     |
| Command + Shift + Up/Down   | *当前语句上下移动（语法允许范围内）                          |
| Command + option + Up/Down  | 当前内容强制上下移动                                         |
| Command + Shift + U         | 大小写切换当前选中内容或者当前行                             |
| Command + Shift + Enter     | *语句完成，或换行                                            |
| Option + Shift + Click      | 在此处添加或删除一个光标（可多行复制/粘贴）                  |
| Option + Shift + Left/Right | *按词边界选中内容                                            |
| Command + O                 | 全局查找类或接口                                             |
| Command + F                 | *当前文档查找文本                                            |
| Command + Shift + F         | 文件夹全局搜索                                               |
| Command + R                 | 当前文档全局替换文本                                         |
| Shift + Shift               | *全局搜索                                                    |

### 重构代码

| 按键                 | 说明                            |
| -------------------- | ------------------------------- |
| Controll + T         | 当前位置打开重构窗口            |
| Option + Command + M | *抽离选中代码至方法块           |
| Option + Command + P | 将方法内局部变量抽取至方法参数  |
| Option + Command + F | *字段抽离，局部变量提至全局变量 |
| Option + Command + V | 变量抽离，快速生成对象变量      |

> 带*的表示非常常用的快捷键