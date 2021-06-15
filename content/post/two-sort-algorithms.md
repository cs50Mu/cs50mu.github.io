+++
title = "冒泡排序和插入排序"
date = "2014-03-30"
slug = "2014/03/30/two-sort-algorithms"
Categories = []
+++

## 冒泡排序
应该是最简单的吧，但是效率很低。对于给定的n个无序数列，不断的比较相邻的两个数的大小，若满足条件，则互换两个数的位置，这样经过一次循环后，最大的（或者最小的）数就冒出来了，经过n次循环，顺序就排好了。具体伪代码如下：

        for(i=0; i < n; i++) {
                for(j = 0; j < n - i - 1; j++) {
                        if(target[j] > target[j+1]) {
                                temp = target[j+1];
                                target[j+1] = target[j];
                                target[j] = temp;
                        }   
                }   
        }   

## 插入排序
模仿的是打牌的时候，插牌的方法。对于给定的n个无序数列，从第二个数开始往下循环到第n个数，在处理某个数的时候，依次比较这个数跟它之前的数的大小，直到找到合适的位置后就插入。具体伪代码如下：

        for(i = 1; i < n; i++) {
                key = target[i];  // 待插的值
                for(j = i - 1; (j >= 0) && (target[j] > key); j--) {
                        target[j+1] = target[j];   // 移位，给要插入的数让位
                }
                target[j+1] = key;    // 插进去
        }

