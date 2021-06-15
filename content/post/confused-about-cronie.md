+++
title = "confused about cronie"
date = "2013-10-08"
slug = "2013/10/08/confused-about-cronie"
Categories = ["linux", "cron "]
+++

昨天想要为每日一图脚本添加定时运行时才发现，貌似crond不工作了。    
于是发现原来的dcron早就被踢进AUR了，现在的crond是cronie童鞋。。可是要肿么用呢？

发现Archlinux论坛上也有人有这样的疑问，思路整理如下：

目前，可以使用`crontab -e`来添加命令或者把想定时运行的命令放在相应的`cron.{hourly,daily,monthly,weekly}`文件夹中。两者的区别是，通过`crontab -e`添加的任务可以指定具体的运行时间，可以精确到分钟，但需要保证机器一直运行，若在指定的时间点计算机没有在线，那么任务就不会运行；而放在那几个文件夹的脚本不可以指定精确的运行时间，但不要求计算机一直在线，计算机一旦在线就会检查相应文件夹下的脚本是否应该执行，若符合执行的条件则执行。那么它是通过什么方法来检查的呢？配置文件`/etc/anacrontab`的一个字段`period in days`指定了某个文件夹下的脚本的运行间隔时间，若计算机看到某个脚本距上次执行时间已经有n天或大于n天，那么它就会运行这个脚本。    
以下的专业解释来自Man，哈哈    
> Anacron  is used to execute commands periodically, with a frequency specified in days.  Unlike cron(8),it does not assume that the machine is running continuously.  Hence, it can be used  on  machines  that are not running 24 hours a day to control regular jobs as daily, weekly, and monthly jobs.

> Anacron  reads  a  list  of jobs from the /etc/anacrontab configuration file (see anacrontab(5)).  This file contains the list of jobs that Anacron controls.  Each job entry specifies a  period  in  days, a delay in minutes, a unique job identifier, and a shell command.

> For each job, Anacron checks whether this job has been executed in the last n days, where n is the time period specified for that job.  If a job has not been executed in n days  or  more,  Anacron  runs  the job's shell command, after waiting for the number of minutes specified as the delay parameter.

> After  the  command exits, Anacron records the date (excludes the hour) in a special timestamp file for that job, so it knows when to execute that job again.

To summerize then,if you want to execute a script at a particular time, you should use `crontab -e`.If you just want the script to execute regardless of the time, you should drop it in the appropriate folder.

做完上面这些后，我悲催的发现crond其实不适合用于定时运行每日一图的脚本。。最后加在awesome的开机启动，通过`theme.wallpaper`参数可以来设置墙纸，不过有一个矛盾，只能先设置墙纸然后才能下载每日一图。。暂时没找到很好的可以先下载再设置的方法。所以，目前用不了当天的图，用的是前一天的图。。
