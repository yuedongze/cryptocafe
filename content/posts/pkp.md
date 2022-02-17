---
title: "Permutated Kernel Problem"
date: 2022-02-16T16:21:14-08:00
draft: true
math: true
---

# PKPDSS: Permuted-Kernel-Problem-based Digital Signature Scheme

Based on: PKP-IDS, Permutation based identification protocol.

Idea: PKP is a problem where one is give a random matrix $A$ and a vector $v$. Then they are asked to find a permutation $\pi$ such that $A v_\pi = 0$, i.e. $v_\pi$ is the right kernel of $A$.

Finding such permutation $\pi$ is an NP-hard problem. However, we could use a 5-round public-coin interactive protocol to prove knowledge of the secret permutation $\pi$.

We will try to explore how PKP-IDS works, and then follow [Beullens et. al. 2018](https://www.esat.kuleuven.be/cosic/publications/article-3103.pdf) to compress PKP-IDS into PKP-DSS.


## Keygen

The keygen of PKP-IDS is the following.

First, we need to sample a random matrix $A$ and a secret permutation $\pi$. Then we need to find a right kernel $w = Ker(A)$ such that $Aw = 0$. We apply the inverse permutation on $w$ and get $v = w_{\pi^{-1}}$. Then our public key is simply $(A, v)$ and our secret key is the permutation $\pi$.

A few interesting observations that [Beullens et. al. 2018](https://www.esat.kuleuven.be/cosic/publications/article-3103.pdf) has pointed out are:

- With high probability, we can convert a random matrix $A$ of dimension $m \times n$ into form $[I_m \vert A']$ where $A' \in \mathbb{F}_p^{m \times n-m}$. Then we no longer need to memorize the left hand side $I_m$ identity matrix so we can reduce the communication cost.

- Instead of applying Gaussian Elimination to $A$ in order to find its right kernel $w$, we could do the other way around. First, we randomly sample vector $v$ and every column but the last for matrix $A$. We leave the last column of matrix $A$ blank. We then sample a random permutation $\pi$ and set $w = v_\pi$. Essentially, we get something like:

$$
\begin{bmatrix}
   a_{0,0} & a_{0,1} & a_{0,2} & \dots & a_{0,n-2} & a_{0,n-1}^* \\\\
   \vdots & \ddots & \ddots & \ddots & \ddots & \vdots \\\\
   a_{m-1,0} & a_{m-1,1} & a_{m-1,2} & \dots & a_{m-1,n-2} & a_{m-1,n-1}^*
\end{bmatrix}
\begin{bmatrix}
w_0 \\\\
w_1 \\\\
\vdots \\\\
w_{n-1}
\end{bmatrix}
= 0
$$

Where $a_{i,j}$ is the i-th row and j-th column of matrix $A$ and $a_{i,n-1}^*$ is the blank column.

We can see that, for each row $i$, we can make $\sum_{j=0}^{n-1} a_{i,j} w_j = 0$. Since $a_{i,n-1}^*$ is left uncomputed, we can simply compute the inverse and obtain:

$$a_{i,n-1}^* = \left(-\sum_{j=0}^{n-2} a_{i,j} w_j \right) \cdot w_{n-1}^{-1}$$

Note $w^{-1}_{n-1}$ can be computed once and stored in memory.

Then, it's obvious to show that:

$$
\begin{align*}
\sum_{j=0}^{n-1} a_{i,j} w_j &= a_{i,n-1}^* \cdot w_{n-1} + \sum_{i=0}^{n-2} a_{i,j} w_j\\\\
&= \sum_{j=0}^{n-2} a_{i,j} w_j - \sum_{j=0}^{n-2} a_{i,j} w_i\\\\
&= 0
\end{align*}
$$


Therefore, we can do the dimension reduction trick together with the sampling trick, and obtain the key materials required for the IDS protocol.


## Identification Scheme
After KeyGen, we are given a public key $pk = (A, v)$ and a secret key $sk = \pi$. The next step is to perform the actual PKP-IDS scheme to achieve a zero-knowledge proof that Alice has knowledge of the secret $\pi$.

The entire protocol has 5 rounds:

1. First, Alice sample a random permutation $\sigma \in S_n$ and a random vector $r \in \mathbb{F}_p^n$. Then Alice generates a commitment of the following two items:
  - $C_0 \leftarrow Com(\sigma, Ar)$, which is the commitment of matrix $A$ blinded by random vector $r$ using $\sigma$ as randomness.
  - $C_1 \leftarrow Com(\pi\sigma, r_\sigma)$, which is the commitment of random vector $r$ (permuted by $\sigma$) using $\pi\sigma$ as randomness.

2. Upon receiving both commitments, Bob generates a random challenge $c \in \mathbb{F}_p$ and sends back $c$.

3. Alice computes $z \leftarrow r_\sigma + cv_{\pi\sigma}$ and sends $z$.

4. Bob generates a random coin flip $b \leftarrow \{0,1\}$ and sends $b$.

5. Alice sends $\sigma$ if $b=0$ or $\pi\sigma$ if $b=1$.

6. If $b=0$, Bob can compute the first commitment himself by $C^* \leftarrow Com(\sigma, Az_{\sigma^{-1}})$. If $b=1$, Bob can compute the second commitment himself by $C^* \leftarrow Com(\pi\sigma, z - cv_{\pi\sigma})$. Bob checks if the commitment he computes matches the original commitment sent from Alice and outputs Yes/No.

We use a hash function to model the commitment scheme here. Given a hash function $H$:

$$Com(r, m) \leftarrow H(H(r) || H(m))$$


## From Identification Scheme to Signature Scheme

PKP-IDS is a scheme is a public-coin protocol. This means that the only thing that Bob provides during the protocol is nothing but randomly sampled bits. This implies that we could use Fiat-Shamir heuristic to compress the interactive protocol into a non-interactive protocol by producing Bob's challenge via hashing Alice's transcript.

However, due to the $\frac{p+1}{2p}$ soundness error of the PKP-IDS scheme, we need to repeat this protocol $N$ times in order to achieve a reasonable soundness error.

1. First, Alice generates the $(A, v), \pi$ instance. Then, she samples $\sigma$ and generates the two commitments $C_0, C_1$.

2. Alice is then supposed to send over $C_0, C_1$ to Bob and receive a challenge $c$. Instead of doing that, we use Fiat-Shamir to apply a hash on Alice's transcript and sample the challenge, i.e. $c \leftarrow H(C_0 || C_1)$. A potential strategy would be modding down the randomness bytes into the finite field and take the remainder.

3. Upon deriving a challenge $c$, Alice will then compute the response $z$ using the derived $c$.

4. Following the same idea of Step 2, Alice can now use the combination of the transcript to derive the next challenge $b$, i.e. $b \leftarrow H(C_0 || C_1 || z)$.

5. Lastly, Alice answers the second challenge with either $\sigma$ or $\pi_sigma$ and sends the entire transcript to Bob.

$$
\mathbf{PKP_{IDS}}(A, v, \pi) \rightarrow C_0, C_1, z, \sigma/\pi\sigma
$$

Upon receiving the non-interactive proof, Bob can follow the same steps as Alice to derive the challenge $c$ and $b$ respectively and invoke the same verify function in the interactive protocol to validate the proof.

## Towards a Signature Scheme

Right now, the non-interactive proof above simply proves the knowledge of the secret premutation $\pi$. However, we would want to not only prove that, but also somehow bind this proof with a specific message $m$, such that only the ones who know $\pi$ could generate this proof. This is therefore called digital signature.

How this is typically done is to encode the message $m$ as part of the randomness of the commitment scheme, and later reveal $m$ as part of opening the commitment. In our implementation, it's as simple as putting the message into the transcript before hashing and generating the challenges. In other words:

$$c \leftarrow H(m || C_0 || C_1) \\\\
b \leftarrow H(m || C_0 || C_1 || z)$$