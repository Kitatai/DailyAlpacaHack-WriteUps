# Decrypt Shop

[問題](https://alpacahack.com/daily/challenges/decrypt-shop)

$c$ 以外なら何でも復号してくれるとのことなので、もし $n+c$ を復号してくれれば、

$$
(n+c)^d
= \sum_{i=0}^{d} \binom{d}{i} n^{d-i} c^i
\equiv c^d
\pmod n
$$

となる。$i < d$ の項には $n$ が因数として含まれるため、法 $n$ ではすべて $0$ になる。最後の $i=d$ の項だけが残り、$c^d$ になる。

しかし、入力 $x$ は $0 \leq x < n$ を満たす必要がある。また、`(n+c) % n` は $c$ になってしまうため、そのままでは `x == c` のチェックに引っかかる。

そこで、代わりに $n-c$ を送信する。二項定理より、

$$
(n-c)^d
= \sum_{i=0}^{d} \binom{d}{i} n^{d-i} (-c)^i
\equiv (-c)^d
\pmod n
$$

である。今回の RSA では $e=65537$ で奇数であり、また $ed \equiv 1 \pmod{\varphi(n)}$ である。$\varphi(n)$ は偶数なので、$ed$ は奇数、したがって $d$ も奇数である。

よって、

$$
(-c)^d = -c^d
$$

となる。暗号文 $c$ は平文を $m$ とすると

$$
c \equiv m^e \pmod n
$$

なので、

$$
c^d \equiv m^{ed} \equiv m \pmod n
$$

である。したがって、$n-c$ を復号した結果を $r$ とすると、

$$
r \equiv (n-c)^d \equiv -m \equiv n-m \pmod n
$$

となる。flag の整数値 $m$ は $0 < m < n$ なので、

$$
m = n-r
$$

として復元できる。
