title: Round up to the next highest power of 2
category: snippet 
date: 
tags: [snippet]
toc: false
comments: false
---
#### version1

```bash
static unsigned int power2gt(unsigned int size) {
  while (1) {
    if (i >= size) return i;
    i *= 2;
  }
}
```

#### version2

```bash
static unsigned int power2gt(unsigned int size) {
    --size;
    size |= size >> 1;
    size |= size >> 2;
    size |= size >> 4;
    size |= size >> 8;
    size |= size >> 16;
    return size + 1;
}
```
#### 参考资料
1. the explanation of version2:[here](http://bits.stephan-brumme.com/roundUpToNextPowerOfTwo.html)
2. [Reidis#issue4776](https://github.com/antirez/redis/issues/4776)
