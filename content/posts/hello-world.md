---
title: "Hello, Crypto"
date: 2022-02-14T22:31:56-08:00
draft: true
math: true
---

Bitcoin?

```
def mine_btc(block):
  while True:
    nonce = random.randbytes(32)
    if hashlib.sha256(block + nonce).digest().hex()[:8] == "00000000":
      print("I'm rich!")
      return
```

Or maybe some $\mathcal{math}$ helps:

$$
e = mc^2
$$
