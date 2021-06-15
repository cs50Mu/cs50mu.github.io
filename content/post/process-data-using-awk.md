+++
title = "process data using awk"
date = "2017-02-22"
slug = "2017/02/22/process-data-using-awk"
Categories = []
+++

###AWK学习笔记
以下内容主要参考自《The AWK Programming Language》一书以及GNU的《The GNU Awk User’s Guide》。

- the structure of an AWK program

		pattern { action }
the basic operation of awk is to scan a sequence of input lines one after another, searching for lines that are matched by any of the patterns in the program. For each pattern that matches, the corresponding action is performed.

- simple example
	- NF, the number of fields. Print the number of fields and the first and last fields of each input line.

			{ print NF, $1, $NF }
			
	- NR, the number of lines read so far. Prefix each line of the file with its line number.

			{ print NR, $0 }
			
	- Putting text in the output.

			{ print "this is",NR,"line",$0 }
			
	- Fancier Output, use `printf` statement
	- Selection by comparision

			$2 >= 5
			
	- Selection by text content

			$1 == "Susie"
			
	- Combinations of patterns. `&&`,`||`,`!`, which stand for AND, OR, NOT

			$2 >=4 || $3 >= 20
			
	- BEGIN and END. The special pattern `BEGIN` matches before the first line of the first input file read, and `END` matches after the last line of the last file has been processed. The following example uses `BEGIN` to print a heading.

			BEGIN { print "NAME  RATE  HOURS"; print "" }
			{ print }
			
		注意，第一个`print ""` 输出一个空行
	- Exchange the first two fields of every line and then print the line

			{ temp = $1; $1 = $2; $2 = temp; print }
	- Print in reverse order the fields of every line

			{ for (i = NF; i > 0; i = i - 1) printf("%s ", $i)
				printf("\n")
			}
	- Print the sums of the fields of every line

			{ sum = 0
				for (i = 1; i <= NF; i = i + 1) sum = sum + $i
				print sum
			}
	- Print every line after replacing each field by its absolute value

			{ for (i = 1; i <= NF; i = i + 1) if ($i < 0) $i = -$i
				print
			}
		
- Computing with AWK

	In awk, user-created variables are **not** declared.
	- Counting

			$3 > 15 { emp = emp + 1 }
			END   { print emp, "employees worked more than 15 hours" }
		Awk varibales used as numbers begin life with the value 0, so we didn't need to initialize emp.
		
	- String Concatenation

			{ names = names $1 " " }
			END { print names }
		collects all the employee names into a single string, by appending each name and a blank to the previous value in the variable names.
- Control-Flow Statements

	- if-else statement

			$2 > 6 { n = n + 1; pay = pay + $2 * $3 }
			END    { if (n > 0)
							print n, "employees, total pay is", pay,
										"average pay is", pay/n
					else
						print "no employees are paid more than $6/hour"
					}
					
	- While Statement
	- For Statement	

- String-Matching Patterns

	- `[.]`matches a period and `^[^^]`matches any character except a caret at the beginning of a string		


- String-Manipulation Functions
	- `gsub`
		- Search target for all of the longest, leftmost, nonoverlapping matching substrings it can find and replace them with replacement. The ‘g’ in gsub() stands for “global,” which means replace everywhere.
		- `gsub("^=\"|\"$", "", $33)`  --->  "=1234567" ---> 1234567

- Built-in Variables That Control awk
	- FS
		- The input field separator.

	- OFS
		- The output field separator. It is output between the fields printed by a print statement. Its default value is " ", a string consisting of a single space.
		- 对`print`有效，会被`printf`忽略

	- RS
		- The input record separator. Its default value is a string containing a single newline character, which means that an input record consists of a single line of text.
		- 这个字段的用处是，可以手动指定行结束符，一般用不上，不过当在unix系统上处理windows生成的文件时，并且在某些字段中含有未转义的`\n`时。这时，指定`RS="\r\n"`后，你就会感谢上帝给你提供了RS这个设置
	- ORS
		- The output record separator. It is output between the fields printed by a print statement. Its default value is " ", a string consisting of a single space. 
		- 注意，该变量只对`print`语句有效，`printf`语句会忽略这个变量的设置，**必须显式指定行分隔符**，否则所有的记录都在一行上！
	- FPAT
		- gawk专有的
		- A regular expression (as a string) that tells gawk to create the fields based on text that matches the regular expression. Assigning a value to FPAT overrides the use of FS and FIELDWIDTHS for field splitting. 
		- 处理字段中含有`,`的csv文件的大杀器
			- 比如这样的`Robbins,Arnold,"1234 A Pretty Street, NE",MyTown,MyState,12345-6789,USA`
			- 可以设置`FPAT = "([^,]+)|(\"[^\"]+\")"`来正确提取出相应的字段，否则若只用逗号分隔符的话，会提取出错

- Built-in Variables That Convey Information

	The following is an alphabetical list of variables that awk sets automatically on certain occasions in order to provide information to your program. 
	
	- ARGC, ARGV
		- The command-line arguments available to awk programs are stored in an array called ARGV. ARGC is the number of command-line arguments present.
	- ARGIND
		- gawk专有，在处理多个文件时很有用
		- The index in ARGV of the current file being processed. Every time gawk opens a new data file for processing, it sets ARGIND to the index in ARGV of the file name. When gawk is processing the input files, ‘FILENAME == ARGV[ARGIND]’ is always true. 

				awk -F, 'ARGIND==1{data[$1]=1} ARGIND==2{if(!($1 in data)) print $1}' dedao_mall_transaction_uniq.csv gongzhonghao_caifutong_uniq.csv > in_caifutong_not_in_dedao.csv
				
			上面这个例子可用于比较两个文件的差异
			
- Array
	- [https://www.gnu.org/software/gawk/manual/gawk.html#Array-Basics](https://www.gnu.org/software/gawk/manual/gawk.html#Array-Basics)

			awk -F, '{ data[$1]["amount"] += $5; data[$1]["product_price*amount"] += $3*$5; data[$1]["refund"] += $6; data[$1]["pay_price*amount"] += $4*$5; data[$1]["discount_price"] += ($3-$4)*$5;}\
		    END{for (d in data) printf("%s,%d,%.2f,%.2f,%.2f,%.2f\n", d, data[d]["amount"], data[d]["product_price*amount"], data[d]["pay_price*amount"], data[d]["discount_price"], data[d]["refund"])}' \
		    tt_new.csv | awk -F, 'BEGIN{printf("%s,%s,%s,%s,%s,%s\n", "sku", "amount", "product_price*amount", "pay_price*amount", "discount_price", "refund")} {print $0}' > summary_by_sku.csv
		    
		   上面的例子用到了gawk扩展的多维array特性，不适用于unix下默认的awk
		   
###避坑指南
这几天用awk写了几个处理数据的小程序，顺便总结下遇到的坑：

- 由于操作系统平台不同引起的问题
	- 换行符
		- win平台为`\r\n`，*nix平台为`\n`。因此在比较两个文件时，切记先把换行符转换为一致再比较，否则会发生“明明看起来是一样的，但为啥不一样呢？”的令人困惑的情况。
	- 文件编码问题
		- *nix系统下一般使用`utf-8`编码，win平台中文环境一般为gb18030编码。更需要注意的是中国人民的好朋友Excel，在编辑完csv文件再保存后，会把文件编码强制更改为gb18030，即使原来的文件编码是utf-8.

###参考
- The GNU Awk User’s Guide
    - [https://www.gnu.org/software/gawk/manual/gawk.html](https://www.gnu.org/software/gawk/manual/gawk.html#Index)
- The AWK Programming Language 
