---
date: true
comment: true
totalwd: true
---

# VAE

!!! abstract "简介"
    VAE 是一种生成模型，通过学习数据的**分布**，从而生成新的数据  
    这与 AE 不同，AE 是通过学习数据的**特征**，从而重建数据

## 基本原理

假定图片数据空间为 $X$，潜在空间为 $Z$，VAE 的目标是学习 $p(x)$，即数据的分布
$$
p(x) = \int p(x|z)p(z)dz
$$
其中 $p(x|z)$ 为条件概率，即给定 $z$ 生成 $x$ 的概率；$p(z)$ 为先验概率，即 $z$ 的分布。但由于不可能完全穷举 $z$ 的所有可能，所以我们可以换用变分的角度，即通过后验概率
$$
p(z|x) = \frac{p(x|z)p(z)}{p(x)}
$$
来缩小 $z$ 的范围。这里的 $p(z|x)$ 可看作 VAE 的 Encoder，$p(x|z)$ 可看作 VAE 的 Decoder

## 简略推导

??? example "VAE 图解"
    下图展示了 VAE 的图模型，其中实线表示生成模型 $p_{\theta}(z)p_{\theta}(x|z)$，虚线表示对难解后验 $p_{\theta}(z|x)$ 的变分近似 $q_{\phi}(z|x)$
    \tikzpicture-automata
        \node[state] (x) {$x$};
        \node[state, above of=x] (z) {$z$};
        \node[right of=z] (t) {$\theta$};
        \node[left of=z] (p) {$\phi$};
        \draw   (t) edge[above] (z)
                (z) edge[above] (x)
                (t) edge[above] (x);
        \draw[dashed] (p) edge[above] (z)
                    (x) edge[bend left] (z);
        \draw[rounded corners] (-1, -1) rectangle (1, 3);

根据上述想法，记生成模型（Decoder）参数为 $\theta$，协同优化的变分参数（Encoder）为 $\phi$，我们需要学习分布 $q_{\phi}(z|x)$，使得 $q_{\phi}(z|x)$ 尽可能接近 $p(z|x)$。这里我们引入 KL 散度，即

$$
\begin{aligned}
D_{KL}(q_{\phi}(z|x)||p(z|x)) &= E_{z \sim q_{\phi}(z|x)}[\log\frac{q_{\phi}(z|x)}{p(z|x)}] \\
&= E_{z \sim q_{\phi}(z|x)}[\log q_{\phi}(z|x) - \log p(z|x)] \\
&= E_{z \sim q_{\phi}(z|x)}[\log q_{\phi}(z|x) - \log \frac{p_{\theta}(x|z)p_{\theta}(z)}{p_{\theta}(x)}] \\
&= E_{z \sim q_{\phi}(z|x)}[\log q_{\phi}(z|x) + \log p_{\theta}(x) - \log p_{\theta}(x|z) p_{\theta}(z)] \\
\end{aligned}
$$

由于 $p_{\theta}(x)$ 与 $z$ 无关，所以可以从期望中提出
$$
D_{KL}(q_{\phi}(z|x)||p(z|x)) = \log p_{\theta}(x) + E_{z \sim q_{\phi}(z|x)}[\log q_{\phi}(z|x) - \log p_{\theta}(x|z) p_{\theta}(z)]
$$
据此，我们可以定义 evidence lower bound (ELBO) 作为优化目标

$$
\begin{aligned}
\mathcal{L}(\theta, \phi; x) &= \log p_{\theta}(x) - D_{KL}(q_{\phi}(z|x)||p(z|x)) \\
&= -E_{z \sim q_{\phi}(z|x)}[\log q_{\phi}(z|x) - \log p_{\theta}(x|z) p_{\theta}(z)]
\end{aligned}
$$

我们希望输出图片为原始图片的概率最大，即最大化 $p_{\theta}(x)$（针对 Decoder）；同时希望学习到的分布 $q_{\phi}(z|x)$ 尽可能接近 $p(z|x)$，即最小化 KL 散度（针对 Encoder）。所以我们可以通过最大化 ELBO 来优化 VAE

在此基础上，注意到随机采样存在梯度不可导的问题，所以我们可以通过重参数化技巧来解决这个问题，如假设 $z \sim \mathcal{N}(\mu, \sigma^2)$，则可以通过 $\mu + \sigma \odot \epsilon$ 来采样 $z$，其中 $\epsilon \sim \mathcal{N}(0, 1)$（$\epsilon$ 是从标准正态分布中采样的值，在计算图之外）