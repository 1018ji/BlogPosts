---
title: PHP 7 快速排序源码分析
date: 2019-02-20 18:47:21
categories: PHP
tags:
  - PHP
---

# PHP 7 快速排序源码分析
PHP 排序源码位置为 Zend/zend_sort.c 方法为 zend_sort

# 基本排序思路
```
zend_sort 数据 base、数据长度 nmemb、数据元素长度 siz、对比函数 cmp、交换函数 swp
    while True
        如果元素个数小于 16
            处理 0 ~ 5 个元素的排序
                - 0, 1 直接返回
                - 2 zend_sort_2
                - 3 zend_sort_3
                - 4 zend_sort_4
                - 5 zend_sort_5
                返回
            处理 6 ~ 16 个元素的排序
                zend_insert_sort 插入排序
                返回
        如果元素个数大于 16
            枢轴选取
                如果排序个数大于等于 1024，将五等分点的五个数原地排序
                否则将三等分点的三个数原地排序
            交换第一个元素与枢轴元素
            i 为左指针，指向第二个元素
            j 为右指针，指向最后一个元素
            while True
                循环寻找大于枢轴元素元素，i++，如果 i 等于 j 则跳转至锚点
                j 自减去 1 (因为选取枢轴已排序肯定比枢轴元素大)
                循环寻找小于枢轴元素元素，j--, 如果 i 等于 j 则跳转至锚点
                交换 i、j 的元素
                如果 i 等于 j 则跳转至锚点
        锚点
            讲枢轴元素防止最终位置
            如果左侧元素个数小于右侧元素个数
                递归调用 zend_sort 排序左侧
                数据变为左侧
                数据长度变为左侧长度
            如果左侧元素个数大于右侧元素个数
                递归调用 zend_sort 排序右侧
                数据长度变为右侧长度
```

# 其他重要说明
锚点部分设计采用 尾递归消除 [Tail Call Elimination](https://www.geeksforgeeks.org/quicksort-tail-call-optimization-reducing-worst-case-space-log-n/)，降低排序需要的额外空间为 O(logn)
另外还有一种 编译器尾递归优化 [Tail Call Optimization](http://wiki.c2.com/?TailCallOptimization)，基于编译器减少地柜调用栈过大问题

<!-- more -->

# 代码注释
代码中说明 **Derived from LLVM's libc++ implementation of std::sort.**
其实这个是参照 LLVM sort 优化而来，其实也没啥优化

```
// 排序元素为 2 直接排序
static inline void zend_sort_2(void *a, void *b, compare_func_t cmp, swap_func_t swp) /* {{{ */ {
    if (cmp(a, b) > 0) {
        swp(a, b);
    }
}
/* }}} */

// 排序元素为 3 直接排序
static inline void zend_sort_3(void *a, void *b, void *c, compare_func_t cmp, swap_func_t swp) /* {{{ */ {
    if (!(cmp(a, b) > 0)) {
        if (!(cmp(b, c) > 0)) {
            return;
        }
        swp(b, c);
        if (cmp(a, b) > 0) {
            swp(a, b);
        }
        return;
    }
    if (!(cmp(c, b) > 0)) {
        swp(a, c);
        return;
    }
    swp(a, b);
    if (cmp(b, c) > 0) {
        swp(b, c);
    }
}
/* }}} */

// 排序元素为 4 直接排序
static void zend_sort_4(void *a, void *b, void *c, void *d, compare_func_t cmp, swap_func_t swp) /* {{{ */ {
    zend_sort_3(a, b, c, cmp, swp);
    if (cmp(c, d) > 0) {
        swp(c, d);
        if (cmp(b, c) > 0) {
            swp(b, c);
            if (cmp(a, b) > 0) {
                swp(a, b);
            }
        }
    }
}
/* }}} */

// 排序元素为 5 直接排序
static void zend_sort_5(void *a, void *b, void *c, void *d, void *e, compare_func_t cmp, swap_func_t swp) /* {{{ */ {
    zend_sort_4(a, b, c, d, cmp, swp);
    if (cmp(d, e) > 0) {
        swp(d, e);
        if (cmp(c, d) > 0) {
            swp(c, d);
            if (cmp(b, c) > 0) {
                swp(b, c);
                if (cmp(a, b) > 0) {
                    swp(a, b);
                }
            }
        }
    }
}
/* }}} */

// 插入排序
ZEND_API void zend_insert_sort(void *base, size_t nmemb, size_t siz, compare_func_t cmp, swap_func_t swp) /* {{{ */{
    switch (nmemb) {
        case 0:
        case 1:
            break;
        case 2:
            zend_sort_2(base, (char *)base + siz, cmp, swp);
            break;
        case 3:
            zend_sort_3(base, (char *)base + siz, (char *)base + siz + siz, cmp, swp);
            break;
        case 4:
            {
                size_t siz2 = siz + siz;
                zend_sort_4(base, (char *)base + siz, (char *)base + siz2, (char *)base + siz + siz2, cmp, swp);
            }
            break;
        case 5:
            {
                size_t siz2 = siz + siz;
                zend_sort_5(base, (char *)base + siz, (char *)base + siz2, (char *)base + siz + siz2, (char *)base + siz2 + siz2, cmp, swp);
            }
            break;
        default:
            {
                // 插入排序算法
                char *i, *j, *k;
                char *start = (char *)base;
                char *end = start + (nmemb * siz);
                size_t siz2= siz + siz;
                char *sentry = start + (6 * siz);
                for (i = start + siz; i < sentry; i += siz) {
                    j = i - siz;
                    if (!(cmp(j, i) > 0)) {
                        continue;
                    }
                    while (j != start) {
                        j -= siz;
                        if (!(cmp(j, i) > 0)) {
                            j += siz;
                            break;
                        }
                    }
                    for (k = i; k > j; k -= siz) {
                        swp(k, k - siz);
                    }
                }
                for (i = sentry; i < end; i += siz) {
                    j = i - siz;
                    if (!(cmp(j, i) > 0)) {
                        continue;
                    }
                    do {
                        j -= siz2;
                        if (!(cmp(j, i) > 0)) {
                            j += siz;
                            if (!(cmp(j, i) > 0)) {
                                j += siz;
                            }
                            break;
                        }
                        if (j == start) {
                            break;
                        }
                        if (j == start + siz) {
                            j -= siz;
                            if (cmp(i, j) > 0) {
                                j += siz;
                            }
                            break;
                        }
                    } while (1);
                    for (k = i; k > j; k -= siz) {
                        swp(k, k - siz);
                    }
                }
            }
            break;
    }
}

ZEND_API void zend_sort(void *base, size_t nmemb, size_t siz, compare_func_t cmp, swap_func_t swp)
{
    while (1) {
        if (nmemb <= 16) {
            // 排序个数小与 16 时采用插入排序，并对 0-5 进行专门优化
            zend_insert_sort(base, nmemb, siz, cmp, swp);
            return;
        } else {
            // 排序个数大与 16 时采用快速排序
            char *i, *j;
            // 第一个元素
            char *start = (char *)base;
            // 最后一个元素
            char *end = start + (nmemb * siz);
            // 确定中间位置
            size_t offset = (nmemb >> Z_L(1));
            // 获取中间元素数值 后期将变为枢轴数值
            char *pivot = start + (offset * siz);

            // 此处为快排优化 三数取中 或者 五数取中
            if ((nmemb >> Z_L(10))) {
                // 排序数值大于 1024 讲五等分点的元素进行排序
                size_t delta = (offset >> Z_L(1)) * siz;
                zend_sort_5(start, start + delta, pivot, pivot + delta, end - siz, cmp, swp);
            } else {
                // 排序数值大于 1024 讲三等分点的元素进行排序
                zend_sort_3(start, pivot, end - siz, cmp, swp);
            }
            // 交换第一个元素数值与中间元素数值交换
            swp(start + siz, pivot);
            // 枢轴 使用第一个元素数值
            pivot = start + siz;
            // i 为第二个元素数值 (开始)
            i = pivot + siz;
            // j 为最后一个元素数值 (结束)
            j = end - siz;
            while (1) {
                while (cmp(pivot, i) > 0) {
                    // 循环获取大于枢轴数值的值
                    i += siz;
                    if (UNEXPECTED(i == j)) {
                        // 退出条件
                        goto done;
                    }
                }
                // j 已知比枢纽大 直接跳过
                j -= siz;
                if (UNEXPECTED(j == i)) {
                    // 退出条件
                    goto done;
                }
                while (cmp(j, pivot) > 0) {
                    // 循环获取大于枢轴数值的值
                    j -= siz;
                    if (UNEXPECTED(j == i)) {
                        // 退出条件
                        goto done;
                    }
                }
                // 交换两个元素
                swp(i, j);
                i += siz;
                if (UNEXPECTED(i == j)) {
                    // 退出条件
                    goto done;
                }
            }
done:
            // 讲枢纽数值放置最终位置 (i - siz) 为数轴的位置
            swp(pivot, i - siz);

            // 递归排序 尾递归消除(Tail Call Elimination) 空间复杂度姜维 input + O(log n)
            // 注意这玩意不是 尾递归优化(Tail Call Optimization) 此种基于编译器(compiler)
            // (左侧数据个数 * siz) < (右侧数据个数 * siz)
            // 尽量将较少部分使用递归排序，可以快速返回
            if ((i - siz) - start < end - i) {
                // 左侧比右侧少 左侧排序
                zend_sort(start, (i - start)/siz - 1, siz, cmp, swp);
                base = i; // 数据元素 自循环处理右侧部分
                nmemb = (end - i)/siz; // 元素数量 自循环处理右侧部分
            } else {
                // 左侧比右侧多 右侧排序
                zend_sort(i, (end - i)/siz, siz, cmp, swp);
                nmemb = (i - start)/siz - 1; // 较少元素数量 自循环处理左侧部分
            }
        }
    }
}
```

-----------------
参考资料
1. 解读STL里的std::sort函数 [点我](https://blog.0xbbc.com/2017/01/analysis-of-std-sort-function/#intro)

-----------------

The END！