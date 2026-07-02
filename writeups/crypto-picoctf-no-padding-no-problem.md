# Crypto - picoCTF《No Padding, No Problem》题解

## 题目信息

- 类别：Crypto
- 难度：中等
- 题目类型：RSA 选择密文攻击
- 核心考点：教材式 RSA 的乘法同态、未填充 RSA 的安全缺陷

## 题目分析

这道题的名字已经给了很强的提示：**没有 Padding 的 RSA 往往是脆弱的**。题目通常会给出公钥参数 `(n, e)`、一段密文 `c`，并提供一个解密接口，但限制你不能直接提交原始密文。

如果是标准的 textbook RSA，那么它满足下面这个性质：

```text
Enc(m1) * Enc(m2) mod n = Enc(m1 * m2 mod n)
```

也就是说，RSA 在没有填充时具有**乘法同态**。这正是漏洞所在。

## 解题思路

目标是绕过“不能直接解原密文”的限制，构造一个**相关密文**，让服务端替我们解出一个和原文有关的值，再把原文还原出来。

设原始密文为：

```text
c = m^e mod n
```

任选一个和 `n` 互素的整数 `s`，计算：

```text
c' = c * s^e mod n
```

那么服务端解密 `c'` 得到的是：

```text
m' = (m * s) mod n
```

因为 `s` 是我们自己选的，所以只要能算出 `s` 在模 `n` 下的逆元 `s^-1`，就可以恢复原文：

```text
m = m' * s^-1 mod n
```

## 具体步骤

1. 获取题目给出的 `n`、`e`、`c`
2. 选择一个较小且和 `n` 互素的 `s`，例如 `s = 2`
3. 计算伪造密文 `c' = c * pow(s, e, n) % n`
4. 把 `c'` 发给解密 oracle
5. 得到返回值 `m'`
6. 计算 `s` 的逆元 `inverse(s, n)`
7. 还原原始明文 `m = m' * inverse(s, n) % n`
8. 把整数形式的明文转成字节串，即可拿到 flag

## Exploit 示例

```python
from Crypto.Util.number import long_to_bytes, inverse

n = 0xD6C7B1F3A1B5C2D7E9F1
e = 65537
c = 0x6A2F7C91D3E8A4

s = 2
c_fake = (c * pow(s, e, n)) % n

# 这里假设 decrypt_oracle(c_fake) 会返回服务端对 c_fake 的解密结果
m_fake = decrypt_oracle(c_fake)

m = (m_fake * inverse(s, n)) % n
flag = long_to_bytes(m)
print(flag)
```

如果是远程题，`decrypt_oracle` 一般就是一次菜单交互或者一次 socket 发送。

## 为什么这个攻击成立

原因在于 textbook RSA 只保证了**数学上的可逆性**，却没有提供现代公钥加密需要的**语义安全性**。一旦缺少 OAEP 之类的随机填充，攻击者就能利用其代数结构构造“相关密文”。

这和现代 RSA 的使用方式差别很大：

- 现代公钥加密通常使用 `RSA-OAEP`
- 现代签名通常使用 `RSA-PSS`
- 直接对消息做 `m^e mod n` 几乎总是不安全的

## 这题的关键坑点

- 不能忘记检查 `gcd(s, n) = 1`
- 还原明文时要用模逆，不是普通除法
- 明文一般是大整数，最后要转换成字节串再看 flag
- 有些题会限制某些输入格式，需要注意十进制、十六进制或字符串传参方式

## 知识点总结

这道题非常适合用来理解**“为什么 RSA 不能裸用”**。它表面考的是 RSA，实质考的是：

- textbook RSA 的乘法同态
- 选择密文攻击的基本思路
- 模逆运算
- 现代填充方案的重要性

如果做完这题能牢牢记住一句话，那就是：

> **没有安全填充的 RSA，不是现代意义上安全的加密方案。**
