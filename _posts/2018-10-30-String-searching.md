---
layout: post
title: "String searching"
subtitle: "定位字符串的方法有几何？"
date: 2018-10-30 21:38
tags: 
    - Algorithm
---

# Question

给定两个字符串 $M,N,\lvert M\rvert\ge\lvert N\rvert$ ，找出子串 $N$ （又叫模式串，pattern）出现在 $M$ 中的位置。

# Solution

## Enumeration

- 枚举所有来自 $M$ 的长度为 $\lvert N\rvert$ 的子串和 $N$ 逐一比对。
- 具体方法为把 $M$ 的每一个字符 $M[i]$ 当作起点，比较子串 $M[i:i+n]$ 和 $N$。
- 最多比较 $m-n+1$ 个子串，而每次至多比对 $n$ 个字符，因此最坏情况下，时间复杂度为 $O((m-n+1)n)=O(mn)$

```cpp
/**
 * src: null-terminated string, M
 * sub: null-terminated string, N, len(M) >= len(N)
 */
const char* find_substring(const char* src, const char* sub) {
    auto len = strlen(sub);    // n
    auto end = src + strlen(src) - len;
    while(src != end && strncmp(src++, sub, len) == 0);
    return src;
}

```

## Longest common substring

这个问题可以直接套用求两个字符串的最长公共子串的方法。在最长公共子串问题中，令 $f(i,j)$ 表示子串$M[:i]$ 和 $N[:j]$ 的 **公共后缀** 的长度。

$$
f(i,j)=\begin{cases}
    f(i-1,j-1)+1 &, M[i]=N[j]\\
    0 &, M[i]\not=N[j]
\end{cases}
$$

最长公共子串的长度就是上述公共后缀的最大值。

空间复杂度为 $O(mn)$，使用滚动数组可优化为 $O(n)$。时间复杂度总是 $O(mn)$。

```cpp
const char* find_substring(const char* M, const char* N) {    
    auto m = strlen(M), n = strlen(N);
    std::vector<int> roll[2] = {
        std::vector<int>(n),
        std::vector<int>(n)
    };

    auto prev = roll[0].data(), curr=roll[1].data();
    auto retpos = M + m;
    auto postlen = -1;

    for(auto j=0;j<n;++j) {
        if (M[0] == N[j]) {
            prev[j] = 1;
        }
    }

    for(auto i=1;i<m;++i) {
        if (M[i] == N[0]) {
            curr[0] = 1;
        }

        for(auto j=1;j<n;++j) {
            if (M[i] == N[j]) {
                if (M[i-1] == N[j-1]) {
                    curr[j] = prev[j-1] + 1;
                } else {
                    curr[j] = 1;
                }
            }

            if (curr[j] > postlen) {
                postlen = curr[j];
                retpos = M + i + 1 - postlen;
            }
        }

        std::swap(curr, prev);
    }

    return retpos;
}
```

## KMP

### Search

第一个方法枚举子串，其实可以看作每次比对子串 $M[i:i+n]$ 失败后，将起点 $i$ 往前移动了一个位置 $i+1$ ，下一次从这个位置开始比对新的子串 $M[i+1:i+n+1]$。这种方法在匹配失败之后将起点往后移动 $1$ 个位置，但是现在想要提高效率而实现以下目标：

1. $i$ 向后移动更长的距离；
2. 减少重复比对 $M$ 中字符的次数。

上述两个目标其实可以用同一个方法做到。用 $M'$ 表示子串 $M[i:i+n]$。$M'[j]$ 字符匹配失败意味着前面 $j$ 个字符(即子串 $M'[:j]$ )匹配成功；在这个时候如果可以将 $i$ 移动 $j-k$ 个距离，就意味着 $M'[:j]$ 的**真后缀**(proper suffix) $M'[-k:]$ 和**真前缀**(proper prefix) $M'[:k]$ 相同。

$$
\begin{array}{}
    M'[:j+1]\Rightarrow& C_0C_1C_2\cdots C_{k-1}C_k\cdots\overbrace{C_{j-k}C_{j-(k-1)}\cdots C_{j-1}}^{\text{Postfix of length }k}\cdot C_j\\
    \\
    N[:j+1]\Rightarrow & \underbrace{A_0A_1A_2\cdots A_{k-1}}_{\text{Prefix of length } k}A_k\cdots \underbrace{A_{j-k}A_{j-(k-1)}\cdots A_{j-1}}_{\text{Postfix of length }k} \cdot A_j\\
\end{array}
$$

$$
\footnotesize{M'[:j+1] \text{ differs from } N[:j+1] \text{ at }j\text{-th character only.}}
$$

$$
\begin{aligned}{}
    C_0C_1\cdots&\overbrace{C_{j-k}C_{j-(k-1)}\cdots C_{j-1}}^{\text{Postfix of length }k}\cdot C_j\\
    \\
    &\underbrace{A_0\cdot A_1\cdot A_2\cdots A_{k-1}\cdot}_{\text{Prefix of length } k}A_k\cdots A_{j-1}A_j\\
\end{aligned}
$$

$$
\footnotesize{\text{That means a postfix of }N[:j]\text{ is also a prefix of length }k.}
$$

反过来说，当字符 $M[i+j]$ 匹配失败，将起点 $i$ 向后移动 $j-k$ 个单位长度；$N[:k]$ 是子串 $N[:j]$ 的某个真前缀，同时也是这个子串的真后缀。同时为了不错过潜在的匹配的子串，要求 $k$ 是满足前面条件的 **最大值** 。

按照这种方法可以让起点 $i$ 移动更长的距离，而不是固定的 $1$ 个单位长度。更为重要的是，起点 $i$ 移动的单位长度可以只由模式 $N$ 确定，这意味着可以在比对字符之前先计算好针对 $N$ 的每个位置的 $k$ 的值。

### Prefix and postifx

用 $P[i]$ 表示以 $S[i]$ 结尾的(真后缀)、同时也是 $S$ 的真前缀的子串的最大长度，即上文提到的 $k$ 值。当前要计算是 $P[i]$ ，考虑字符串前缀 $S[:i]$

$$
C_0C_1\cdots C_{i-1}
$$

已知 $P[i-1]=k_0$，即

$$
\overbrace{C_0C_1\cdots C_{k_0-1}}^{\text{Prefix of length } k_0}C_{k_0}\cdots \overbrace{C_{i-k_0}\cdots C_{i-1}}^{\text{Postfix of length }k_0}
$$


考察字符 $C_{k_0}$ 和 $C_{i}$：假如相等，那么令 $P[i]=k_0+1$ 就完事了；假如二者不相等，意味着 **不存在长度为$k\ge k_0+1$ 同时又满足条件的后缀**，这是由 $P$ 的定义所决定的；否则，可以模仿证明“某个问题可以用动态规划解决”，用“剪切和粘贴”(cut-and-paste)方法用这个 $k$ 值“覆盖” $k_0$。回到原来的问题上，下一步要考虑的字符串变成 $S[:k_0]$，

$$
\begin{array}{}
    C_0C_1\cdots C_{k_0-1}
\end{array}
$$

而且已知 $P[k_0-1]=k_1$，

$$
    \overbrace{C_0C_1\cdots C_{k_1-1}}^{\text{Prefix of length }k_1}C_{k_1}\cdots \overbrace{C_{k_0-k_1}\cdots C_{k_0-1}}^{\text{Postfix of length }k_1}
$$

当前需要考察字符 $C_{k_1}$ 和 $C_i$。假如二者相等，那么令 $f(S)=k_1+1$，计算结束；否则 **不存在长度大于 $k_1$ 同时又满足条件的后缀** 。到目前为止，这个问题已经具有和上一步骤类似的形式，因此可以归纳出计算 $P$ 的一般办法：

1. $k=i$
2. $k=P[k-1]$
3. If $S[k]$ equals $S[-1]$, output $k+1$ and exit. 
4. If $k=0$, output $k$ and exit.
5. Otherwise, go to step 2.

这个计算 $P[i]$ 的过程可以看作不断尝试（“跳跃式地”）缩小真前缀（的长度），直到这个真前缀紧跟着的后一个字符能够正确匹配。使得这个方法行之有效的关键之处在于，每次缩小真前缀后，仅仅检查 **一个字符** 就能确定是否也是一个真后缀。

### Partial Match Table

结合前面对算法的分析，在 $M[i+j]$ 和 $N[j]$ 匹配失败的时候，下一步匹配的起始位置 $i$ 变为 $i+j-P[j-1]$。由于给定了模式 $N$ 之后 $j-P[j-1]$ 是固定不变的，这部分可以事先计算。

### Implementation

```cpp
const char* find_substring(const char* M, const char* N) {
    if (*N == '\0') {
        return 0;
    }
    auto len = strlen(N);
    std::vector<int> pmt(len);

    for(auto i=1;i<pmt.size();++i) {
        auto k = i;
        while(true) {
            k = pmt[k-1];
            if(N[k] == N[i]) {
                k = k + 1;
                break;
            } else if (k == 0) {
                break;
            }
        }
        pmt[i] = k; 
    }
    
    for(auto i=pmt.size()-1;i>0;i--) {
        pmt[i] = i - pmt[i-1];
    }
    pmt[0] = 1;

    auto i = 0;
    while(*M != '\0' && i < len) {
        if (M[i] == N[i]) {
            i++;
        } else {
            M += pmt[i];
            if (i >= pmt[i]) {
                i -= pmt[i];
            }
            
        }
    }

    return M;
}
```

## Rolling hash

首先，哈希函数 $H(N)$

$$
H(N)=A_0\cdot B^{n-1}+A_1\cdot B^{n-2}+\cdots+A_{n-2}\cdot B^1+A_{n-1}\cdot B^{0}
$$

同样地，对字符串 $M$ 的具有同样长度的子串 $M'_0$ 应用 $H(S)$

$$
H(M'_0)=C_0\cdot B^{n-1}+C_1\cdot B^{n-2}+\cdots+C_{n-2}\cdot B^1+C_{n-1}\cdot B^{0}
$$

因此，匹配子串和模式串变成了比较二者的哈希值；搜索模式串变成了枚举所有子串并比较子串和模式串的哈希值。但重点是上一个子串 $M'_0$ 匹配失败后，下一个子串 $M'_1$ 的哈希值不必从第一个字符 $C_1$ 重新计算，这是因为

$$
\begin{aligned}
    H(M'_1) &= C_1\cdot B^{n-1}+C_2\cdot B^{n-2}+\cdots+C_{n-1}\cdot B^1+C_{n}\cdot B^{0} \\
    &= B\cdot (C_1\cdot B^{n-2}+C_2\cdot B^{n-3}+\cdots+C_{n-1})+C_{n}\\
    &= B\cdot \big(H(M'_0)-C_0\cdot B^{n-1}\big) + C_n\\
    &= B\cdot H(M'_0) + C_n - C_0\cdot B^n
\end{aligned}
$$

即 $H(M'_1)$ 和 $ B\cdot H(M'_0)+C_n$ 模 $B^n$ 同余

$$
H(M'_1)\equiv B\cdot H(M'_0) + C_n\pmod {B^n}
$$

因为 $H(S)\lt B^n$，这意味着 $H(M'_1)$ 可以从 $H(M'_0)$ 和下一个字符 $C_n$ 计算得到。

假如选择一个整数 $B\gt \max{\{C_i\}}$，那么 $H(M_0')$ 的值就相当于一个 $B$ 进制的整数，计算 $H(M_1')$ 的工作就变成了将 $H(M'_0)$ 向左移动 $1$ 个数位、加上最低位然后扔掉最高位。

下面的代码来自 [LeetCode Implement strStr()](https://leetcode.com/problems/implement-strstr/)，在这里 $B=47$。

```cpp
int strStr(string a, string b) {
    if (!b.size()) return 0;
    if (a.size() < b.size()) return -1;
    unsigned int p = 1;
    unsigned int H = 0;
    unsigned int h = 0; 
    for (int i = 0; i<b.size(); ++i) {
        p *= 47u;
        H = H*47u + (unsigned int) (b[i]-'a'+1); 
        h = h*47u + (unsigned int) (a[i]-'a'+1);
    }
    if (h==H) return 0;
    for (int i = b.size(); i < a.size(); ++i) {
        h = h*47u + (unsigned int) (a[i]-'a'+1);
        h-= p*((unsigned int) (a[i-b.size()]-'a'+1));
        
        if (h==H) return i - b.size() + 1;
    }
    return -1;
}
```

然而，在实际应用中应当注意到哈希值可能会超过整数类型的最大值；对于无符号整数而言将发生上溢，这相当于自动进行了一次取模运算，毫无疑问增加了发生哈希冲突的概率。因此为了防止哈希冲突，需要对具有同一哈希值的字符串进行比较(比如 `strncmp`)。
