---
tags:
  - 基本语法
  - 面试常考
---

`volatile` 表示该变量的值可能在任何时候被外部因素更改，例如硬件设备、操作系统或其他线程，让编译器禁止对其的优化，以确保每次访问变量时都会从内存中读取其值，而不是从寄存器或缓存中读取。作用是避免因为编译器优化而导致出现不符合预期的结果。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

volatile int counter = 0;

void *increment(void *arg) {
    for (int i = 0; i < 100000; i++) {
        counter++;
    }
    return NULL;
}

int main() {
    pthread_t thread1, thread2;

    // 创建两个线程，分别执行increment函数
    pthread_create(&thread1, NULL, increment, NULL);
    pthread_create(&thread2, NULL, increment, NULL);

    // 等待两个线程执行完毕
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);

    printf("Counter: %d\n", counter);

    return 0;
}
```

线程 `thread1` 和 `thread2` 都执行对 `counter` 的 100000 次自增，`volatile` 确保 `counter` 每次都从内存中读取，但是不加锁仍然可能有并发问题。