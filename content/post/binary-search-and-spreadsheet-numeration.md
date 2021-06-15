+++
title = "binary search and spreadsheet numeration"
date = "2014-09-18"
slug = "2014/09/18/binary-search-and-spreadsheet-numeration"
Categories = ["算法"]
+++

昨天去知道创宇的面试失败了，很感谢他们能给我这个机会。通过这次面试，我总算迈出了一步，更加明确了自己应该努力的方向。这次面试反映出的一个很严重的问题就是，手太生，学了这么长时间Python，竟然连一些常用的用法，命令都得现查文档。自己反思一下，没有自己动手写多少代码，这个是问题的关键。   

面试我的雨哥人非常nice，面对我这么菜的表现没有一点不耐烦，他说的一句话让我印象深刻，“既然是有兴趣，平时就多花点时间，对得起兴趣这两个字”。我会的。

## 二分法查找
我想这个应该是最容易理解的算法了吧，我差点就在面试的时候完成了，well,almost...

思路很简单，因为给出的列表已经是排好顺序的了，因此直接找到中间的一个记为mid，若是要找的元素，任务完成。若mid比要找的元素大，好的，那么要找的元素肯定在start跟mid之间了，若mid比要找的元素小，那么要找的元素肯定在mid和end之间了。

```python binary-search
def binarySearch(lst, element, start, end):
    mid = (start + end)/2 
    if element == lst[mid]:
        print 'Found, %d' % mid 
    elif mid == start:
        if element == lst[mid+1]:  # 主要处理两个相邻的数的情况，这种情形下mid必然等于start
            print 'Found, %d' % (mid+1) # 因此如果end（也就是mid+1）不等于element的话，则没找到
        else:
            print 'Not found'
    elif element > lst[mid]:
        return binarySearch(lst, element, mid, end)
    elif element < lst[mid]:
        return binarySearch(lst, element, start, mid)

def binarySearch2(lst, element, start, end):
    mid = (start + end)/2
    if element == lst[mid]:
        return 'Found, %d' % mid 
    elif start > end:
        return 'Not found'
    elif element < lst[mid]:
        return binarySearch2(lst, element, start, mid-1)
    elif element > lst[mid]:
        return binarySearch2(lst, element, mid+1, end)

def binarySearch3(lst, element):
    start = 0 
    end = len(lst) - 1
    while (start <= end):
        mid = (start + end)/2
        if lst[mid] == element:
            return 'Found', mid
        elif lst[mid] > element:
            end = mid - 1
        elif lst[mid] < element:
            start = mid + 1
    return 'Not found'
```


上面的方法一是我自己瞎想的ugly思路，方法三是我在一站式C编程上看到的，方法二是我根据方法三的思路用递归改写的

## spreadsheet numeration
这个题目其实我之前在CodeForces上做过的，可惜没有理解它的真谛，虽然当时误打误撞做出来了，还是没有效果，现在遇到还是不会做，面试中雨哥一句话点醒我，“进制转换”，是啊，这个问题实质上就是十进制转换26进制的问题。那进制转换无非就是要做除法、取余。但是也不是这么简单，有一个比较tricky的问题，就是在转换26的倍数的时候，需要做一些特殊处理才行。

```python spreadsheet-numeration
def numTochar(num):
    result = ''
    ori = num 
    while num:
        mod = num % 26
        if mod == 0:
            mod = 26
            num = num/26 - 1 # special cases for 'Z' 
        else:
            num /= 26
        result = chr(mod + ord('A') - 1) + result
    return result,ori

def numTochar2(num):
    result = ''
    ori = num 
    while num:
        mod = num % 26
        if mod == 0:
            mod = 26
        num -= 1       # a more elegant solution
        num /= 26
        result = chr(mod + ord('A') - 1) + result
    return result,ori
```

方法一是我自己捣鼓的，方法二是我在CodeForces上看到了，效果应该是一样的，都是为了在转换26的倍数的时候能够得出正确的结果，做出的一点特殊处理，不过显然后者更加优雅，每个数在除26之前先减1,这样既对于非26倍数的数没有影响，又能巧妙的处理26的倍数时的特殊情况。
