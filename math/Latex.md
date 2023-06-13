# LateX



## 语法



### 显示样式

* **单行**公式 `$内容$`

  $ f(x) = a + b $

* **多行**公式  `$$ 内容 内容 $$`
  $$
  原函数 \ y = f(x)
  反函数 \ x = f^{-1}(y)
  $$
  



### 排版

| 输入                                                  | 显示                                                         | 含义                              |
| ----------------------------------------------------- | ------------------------------------------------------------ | --------------------------------- |
| `\`                                                   | $\theta \ 中间空格 \ \Theta$                                 | 空格                              |
| `\quad`或`\qquad`                                     | $ 默认间距 a \ b \\ 使用quad \ a \quad b \\ 使用qqaud \ a \qquad b $ | 增大空格间隙                      |
| `\\`                                                  | $\theta \\ \Theta$                                           | 换行                              |
| `\\[nex]`                                             | $ a \\[2ex] b $                                              | 增加垂直间距，默认间距为`\\[1ex]` |
| `{\mathbf 加粗内容}`                                  | ${\mathbf h}$                                                | 加粗                              |
| `\tag{n.m}`                                           | 无法在表格内渲染，在表格下方显示                             | 结尾标号                          |
| `\begin{case}...\end{case}`                           | 在表格下方显示                                               | 分类表达式                        |
| `\begin{array}...\end{array}`，与`\left{ \right.`组合 | 在表格下方显示                                               | 方程组                            |
|                                                       |                                                              |                                   |
|                                                       |                                                              |                                   |
|                                                       |                                                              |                                   |



* **结尾标号**
  $$
  f(x) = a + b \tag1
  $$

  $$
  f(x) = a - b \tag{1.1}
  $$

  

* **分类表达式**
  $$
  f(x)
  \begin{cases}
  \frac 1x, & 如果 \ x \ 是奇数 \\
  3x + 2, & 如果 \ x \ 是偶数 \\
  \end{cases}
  $$
  使用`&` 表示需要对齐的位置

* **多行算式**

$$
\begin{equation}
\begin{split}
f(a) &= b + c - d \\
& \quad + e - f \\
& = g + h \\
& = i
\end{split}
\end{equation}
$$

`begin{equation}` 表示方程开始，`end{equation}` 表示方程结束；`begin{split}` 表示开始多行公式，`end{split}` 表示结束。

* **方程组**
  $$
  \left\{
  \begin{array}{占位}
  a_1x + b_1y - c_1z = d_1 \\
  a_2x + b_2y - c_2z = d_2 \\
  a_3x + b_3y - c_3z = d_3
  \end{array}
  \right.
  $$
  
* Other





### 符号

| 输入                 | 显示                                                     | 含义                                                         |
| -------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| `\geq`               | $\geq$                                                   | 大于等于                                                     |
| `\leq`               | $\leq$                                                   | 小于等于                                                     |
| `>`或`\gt`           | $ \gt \quad > $                                          | 大于                                                         |
| `<`或`\lt`           | $< \quad \lt $                                           | 小于                                                         |
| `=`                  | $=$                                                      | 等于                                                         |
| `\ne`                | $ \ne $                                                  | 不等于                                                       |
| `\not`               | $ \not\gt \not\lt \not\ge \not\le \not= $                | **不**大于/小于/大于等于/小于等于/等于                       |
| `\exist`             | $\exist$                                                 | 存在                                                         |
| `\forall`            | $\forall$                                                | 任意的                                                       |
| `^`                  | $x^2$                                                    | 上标                                                         |
| `_`                  | $x_1$                                                    | 下标                                                         |
| `()`                 | $ f(x) = a + b$                                          | 小括号                                                       |
| `[]`                 | $ x \in[1, 100]$                                         | 中括号                                                       |
| `\{ \}`              | $ \{ (1 - 2) + [ 3 - ( 4 + 5) ]\} $                      | 大括号<br />由于大括号`{}` 被用于分组，因此需要使用`\{`和`\}`表示大括号 |
| `\left(`或`\right}`  | $ \left( \frac{ \frac{1}{2} } { \frac{2}{3} } \right) $  | 自适应括号<br />适用于大中小三种括号类型                     |
| `\sum`               | $ 普通求和 \sum \, 带上下限的求和 \sum_{k = 1}^{n} $     | 求和（累加）                                                 |
| `\prod`              | $ \prod_{n=1}^{10} $                                     | 连乘（累乘）                                                 |
| `\infty`             | $ 正无穷 \infty \, 负无穷 -\infty $                      | 无穷                                                         |
| `\int`               | $ 普通积分 \int \, 带上下限积分  \int_{k = 1}^{\infty} $ | 积分                                                         |
| `\多个i + nt`        | $ \iint \, \iiint \, \iiiint $                           | 多重积分，增加`i`                                            |
| `\bigcup`            | $ \bigcup $                                              | 并集                                                         |
| `\bigcap`            | $ \bigcap $                                              | 交集                                                         |
| `\subset`            | $ \subset $                                              | 包含于/子集（子集合被包含于某个父集合）                      |
| `\subseteq`          | $ \subseteq $                                            | 包含于/子集（子集合被包含于某个父集合）                      |
| `\subsetneq`         | $\subsetneq $                                            | 不包含于/非子集                                              |
| `\supset`            | $ \supset $                                              | 包含/父集                                                    |
| `\in`                | $ \in $                                                  | 属于（元素属于某个集合）                                     |
| `\lim`               | $\lim_{x \to \infty}$                                    | 极限                                                         |
| `\xlongequal`        | $\xlongequal[down]{up}$                                  | 长等于号                                                     |
|                      |                                                          |                                                              |
|                      |                                                          |                                                              |
|                      |                                                          |                                                              |
| `\max`               | $ \max $                                                 | 最大                                                         |
| `\min`               | $ \min $                                                 | 最小                                                         |
| `\langle`或`\rangle` | $ \langle X \rangle $                                    | 尖括号                                                       |
| `\lceil`或`\rceil`   | $ \lceil X \rceil $                                      | 向上取整                                                     |
| `\lflorr`或`\rfloor` | $ \lfloor X \rfloor $                                    | 向下取整                                                     |



### 数学运算

| 输入          | 显示                                                         | 含义                                |
| ------------- | ------------------------------------------------------------ | ----------------------------------- |
| `\frac{1}{2}` | 简单$ \frac 12 $  复杂$ \frac{1}{2} $                        | 分数形式                            |
| `\over`       | $ {1 + 2 + 3 \over 4+5+6} $                                  | 分数形式，使用`\over`分隔组内的内容 |
| `\cfrac`      | $ x = a_0 + \cfrac{1^2}{a_1 + \cfrac{2^2}{a_2 + \cfrac{3^2}{a_4 + ...}} } $ | 连分数                              |
| `\sqrt`       | $ \sqrt a $                                                  | 开根号                              |
|               |                                                              |                                     |
|               |                                                              |                                     |
|               |                                                              |                                     |
|               |                                                              |                                     |
|               |                                                              |                                     |
|               |                                                              |                                     |
|               |                                                              |                                     |
|               |                                                              |                                     |
|               |                                                              |                                     |
|               |                                                              |                                     |
|               |                                                              |                                     |



### 函数

| 输入        | 显示          | 含义     |
| ----------- | ------------- | -------- |
| `\ogn`      | $\log2$       | 对数函数 |
| `\sin空格x` | $\sin x$      | 正弦     |
| `\cos空格x` | $\cos x$      | 余弦     |
| `\tan空格x` | $\tan x$      | 正切     |
| `\arctan x` | $ \arctan x $ | 反正切   |
|             |               |          |
|             |               |          |



### 希腊字母

| 输入       | 小写       | 大写       | 含义    | 音标                       |
| ---------- | ---------- | ---------- | ------- | -------------------------- |
| `\alpha`   | $\alpha$   | $\Alpha$   | alpha   |                            |
| `\bata`    | $\beta$    | $\Beta$    | beta    |                            |
| `\gamma`   | $\gamma$   | $\Gamma$   | gamma   |                            |
| `\delta`   | $\delta$   | $\Delta$   | delta   |                            |
| `\epsilon` | $\epsilon$ | $\Epsilon$ | epsilon | `/'epsɪlɒn/`               |
| `\zeta`    | $\zeta$    | $\Zeta$    | zeta    |                            |
| `\eta`     | $\eta$     | $\Eta$     | eta     |                            |
| `\theta`   | $\theta$   | $\Theta$   | theta   |                            |
| `\iota`    | $\iota$    | $\Iota$    | iota    |                            |
| `\kappa`   | $\kappa$   | $\Kappa$   | kappa   |                            |
| `\lambda`  | $\lambda$  | $\Lambda$  | lambda  |                            |
| `\mu`      | $\mu$      | $\Mu$      | mu      |                            |
| `\nu`      | $\nu$      | $\Nu$      | nu      |                            |
| `\xi`      | $\xi$      | $\Xi$      | xi      | `/ksi/`                    |
| `\omicron` | $\omicron$ | $\Omicron$ | omicron |                            |
| `\pi`      | $\pi$      | $\Pi$      | pi      |                            |
| `\rho`     | $\rho$     | $\Rho$     | rho     |                            |
| `\sigma`   | $\sigma$   | $\Sigma$   | sigma   |                            |
| `\tau`     | $\tau$     | $\Tau$     | tau     | `/taʊ/`或者`/tɔ:/ `        |
| `\upsilon` | $\upsilon$ | $\Upsilon$ | upsilon | `/ˈipsɪlon/`或`/ˈʌpsɪlɒn/` |
| `\phi`     | $\phi$     | $\Phi$     | phi     |                            |
| `\chi`     | $\chi$     | $\Chi$     | chi     | `/kaɪ/`                    |
| `\psi`     | $\psi$     | $\Psi$     | psi     | `/psaɪ/`                   |
| `\omega`   | $\omega$   | $\Omega$   | omega   | `/'əʊmɪɡə/`或`/oʊ'meɡə/`   |


$$
T(n) = 2T(\frac{n}{2})
$$