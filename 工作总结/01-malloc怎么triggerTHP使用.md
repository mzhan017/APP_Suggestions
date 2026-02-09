如果一个小程序频繁的malloc，会造成大页使用，内存被耗光吗？

会，有可能。不是“malloc 本身多吃了内存”，而是 THP用2MB大页来背对时，
容易造成物理内存比逻辑用量多，在内存紧张时更容易把内存耗光。
正常情况
程序频繁 malloc，占用的逻辑内存就是你申请的总和。
THP做的是把本来用4KB小页背对的内存，合并成2MB大页，
理想情况下物理内存总量不会因为“开了 THP”就凭空多一倍。
容易出问题的地方
大页是整2MB一单位的。内核做collapse时，
会对某段2MB的地址空间分配一整颗2MB物理大页。
若这段里其实只有一部分被你的malloc用到（中间有空洞、碎片或已 free 的洞），
就会出现：
逻辑上只用了比如 1MB，但物理上占了一整颗 2MB 大页，多出来的 1MB 就变成内部碎片。
小程序频繁 malloc/free，堆容易碎片化，这种“半满的 2MB”就会变多，
物理内存占用就会比“不用大页时”高，在内存紧张时更容易触发 OOM、把内存耗光。


用这段代码再配合 enabled=always，更容易看到 AnonHugePages。
```
#define _GNU_SOURCE
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
/* 单次 < 2M；>= 128KB 让 glibc 用 mmap，便于内核合并为 THP */
#define CHUNK_SIZE  (128 * 1024)
#define NUM_CHUNKS  32   /* 共 4MB，多块连续匿名页便于 THP collapse */
int main(void) {
    char *ptrs[NUM_CHUNKS];
    size_t i;
    for (i = 0; i < NUM_CHUNKS; i++) {
        ptrs[i] = malloc(CHUNK_SIZE);
        if (!ptrs[i]) {
            fprintf(stderr, "malloc %zu failed\n", i);
            while (i--) free(ptrs[i]);
            return 1;
        }
        /* 触发缺页，让页真正分配并参与 THP 合并 */
        memset(ptrs[i], 0, CHUNK_SIZE);
    }
    /* 给 khugepaged 一点时间做后台 collapse */
    sleep(2);
    printf("pid=%d  total %d * %d = %d KB. Check:\n  cat /proc/%d/smaps | grep -E 'AnonHugePages|Size'\n",
        (int)getpid(), NUM_CHUNKS, (int)CHUNK_SIZE, NUM_CHUNKS * (int)CHUNK_SIZE / 1024, (int)getpid());
    fflush(stdout);
    pause();
    for (i = 0; i < NUM_CHUNKS; i++)
        free(ptrs[i]);
    return 0;
}
```
要点：
多次申请：NUM_CHUNKS 次，每次 CHUNK_SIZE（128KB），总约 4MB，不单次用 2M。
每块都写一遍：memset 让每页 fault in，内核才能把这些匿名页当作 THP 候选。
sleep(2)：让内核后台的 khugepaged 有机会做 collapse，
更容易在 smaps 里看到 AnonHugePages。
先设 always：
echo always > /sys/kernel/mm/transparent_hugepage/enabled
否则没有 madvise 很难有大页。
验证：
#运行程序后另开终端cat /proc/<pid>/smaps | grep -E "AnonHugePages|Size"
若仍看不到 AnonHugePages，可尝试：
把 CHUNK_SIZE 改成 256*1024 或 512*1024（仍 < 2M），
或把 NUM_CHUNKS 调到 64，总内存更大。
确认 enabled 里是 [always]，以及内核配置里打开了 THP。