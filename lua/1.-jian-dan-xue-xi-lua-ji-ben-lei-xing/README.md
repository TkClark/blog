# 1. 简单学习Lua：基本类型

## 1. 8个基本类型

Lua 有8个基础类型,就是下面这样

| 数据类型 | 描述 |
| :--- | :--- |
| nil | 这个最简单，只有值nil属于该类，表示一个无效值（在条件表达式中相当于false）。 |
| boolean | 包含两个值：false和true。 |
| number | 表示双精度类型的实浮点数 |
| string | 字符串由一对双引号或单引号来表示 |
| function | 由 C 或 Lua 编写的函数 |
| userdata | 表示任意存储在变量中的C数据结构 |
| thread | 表示执行的独立线路，用于执行协同程序 |
| table | Lua 中的表（table）其实是一个"关联数组"（associative arrays），数组的索引可以是数字、字符串或表类型。在 Lua 里，table 的创建是通过"构造表达式"来完成，最简单构造表达式是{}，用来创建一个空表。 |

## 2. 怎么判断类型

lua使用跟python一样的方法 type 来做类型判断

```lua
print(type("Hello world"))      --> string
print(type(10.4*3))             --> number
print(type(print))              --> function
print(type(type))               --> function
print(type(true))               --> boolean
print(type(nil))                --> nil
print(type(type(X)))            --> string
```

所有基础类型在使用`type()`方法后返回的结果均是`string`，因此在判断的时候都要使用`""`括起来呦。

```lua
a=1
if ( type(a) == "number" ) then
    print("变量a是number类型")
end
```

## 3. 除了false还有哪些值在判断的时候会被认为是false？

`nil`，`{}`\(空表\)，`false`在判断是会被认为是`false`。

注意注意：0在`lua`中并不代表`false`.

{% hint style="info" %}
接下来的小节只讲一些需要注意的东西

table和function可以直接跳过往后看
{% endhint %}



