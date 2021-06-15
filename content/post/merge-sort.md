+++
title = "归并排序的非递归算法"
date = "2013-07-03"
slug = "2013/07/03/merge-sort"
Categories = ["算法", "merge sort", "python"]
+++

归并排序的效率很高，复杂度只有NlogN，但想起来很费劲啊。。尤其是递归算法，绕来绕去的被绕晕了。。今天在看《Python编程实践》（Practical Programming-An Introduction to Computer Science Using Python）时看到一个归并排序的非递归算法，很好理解，记录下。   
	def merge(L1,L2):
	    """Merge sorted lists L1 and L2 and return the result."""
	    mergedlist = []
	    i1 = i2 = 0 
	    while i1 != len(L1) and i2 != len(L2):
		if L1[i1] <= L2[i2]:
		    mergedlist.append(L1[i1])
		    i1 += 1
		else:
		    mergedlist.append(L2[i2])
		    i2 += 1

	    mergedlist.extend(L1[i1:])
	    mergedlist.extend(L2[i2:])
	    return mergedlist
			
	def mergesort(L):
	    """Sort L."""

	    workspace = []
	    for i in range(len(L)):
		workspace.append([L[i]])

	    count = 0
	    while count < len(workspace)-1:
		newList = merge(workspace[count],workspace[count+1])
		print newList
		workspace.append(newList)
		count += 2

	    return workspace[-1][:]

归并排序的基本思想是这样的，从归并两个字来看我们知道这种算法是通过合并子序列来实现的，普通的乱序序列肯定是不行的，但对于两个已经排好序的序列，很容易把它们合并成一个有序的序列，这就是第一个函数merge的作用。你会问：这有什么用啊？离我们的目标还差很远嘛！我们怎么可能从一个无序的序列变出两个有序的序列嘛？别急，这里用到了二分法的思想（so-called devide and conquer），不断的把问题二分化，直到最简单的情形，在这里也就是单个元素了，两个单个元素肯定是两个有序的序列啊！可以用merge函数来合并它们为一个含有两个元素的有序序列，那么第二个函数mergesort的作用就是：首先把无序列表分解成含有N个含单个元素的小列表，然后依次两两合并这些小列表，并把合并后的有序列表添加到这个列表之后，直到最后合并成一个有序的列表。    

额，表达能力太差了。。还是没有讲清楚   

比如列表原来为[8,7,6,5,4]，那么它的变化为`[[8],[7],[6],[5],[4]]-->[[8],[7],[6],[5],[4],[7,8]]-->[[8],[7],[6],[5],[4],[7,8],[5,6]]-->[[8],[7],[6],[5],[4],[7,8],[5,6],[4,7,8]]-->[[8],[7],[6],[5],[4],[7,8],[5,6],[4,7,8],[4,5,6,7,8]]`   
结果列表的最后一个内嵌列表就是已排好序的序列了。

