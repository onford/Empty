# ChallengeX
建议直接以 lab4_challenge3_shell 的成果为基础，建立 challengeX 分支。
```bash
git checkout -b challengeX lab4_challenge3_shell
```
## Lab1 Tips
### lab1_challenge1
1. 在 lab1_challenge1 的单核单进程环境中，可以直接将当前进程`current`加载时的`elf_ctx`记录下来。现在 challengeX 是多线程，怎么判定当前进程是加载的哪个 ELF 文件？
> 参考做法，在`process`结构中增加一个`elf_name`字段，用于记录当前进程是从哪个路径的 ELF 文件加载得到的。
2. lab1_challenge1 是直地址映射，而现在 challengeX 中用户程序栈帧内部的地址全都是虚拟地址。 
3. 记得在 Makefile 加上编译选项`-fno-omit-frame-pointer`，不然栈里面找不到 fp。
4. 千万千万小心内核爆栈。比如不要在内核的函数里面开太大的数组。

### lab1_challenge2
1. 我习惯性地用`make run`来测试自己的代码，而在这之前最好还是`make clean;make`一下，这样本来不能过的代码可能就过了。即
```bash
make clean;make;make run
```
2. lab2_challenge2 分支的 Makefile 里面，编译选项偷偷加上了`-gdwarf-3`，并未没有在 pke-doc 里面告诉你。我们需要手动在 Makefile 里面加上这个编译选项，以正确生成 .debug_line 段。
```bash
CFLAGS := -Wall -Werror -gdwarf-3 -fno-omit-frame-pointer -fno-builtin -nostdlib -D__NO_INLINE__ -mcmodel=medany -g -Og -std=gnu99 -Wno-unused -Wno-attributes -fno-delete-null-pointer-checks -fno-PIE $(march)
``` 

## Lab2 Tips
### lab2_challenge1
比较简单，略。
### lab2_challenge2
主要注意一点，在进程fork的时候，相应的内存控制块链表也需要进行复制。

## Lab3 Tips
### lab3_challenge1
这个在 lab4_challenge3 里面已经搬运了，因此 challengeX 只需增加用户程序和修改 Makefile 就行。
### lab3_challenge2
也没啥需要注意的，最好可以在用户程序`app_semaphore.c`中加 wait 语句，parent 等一下 child0，child0 等一下 child1。因为 parent 的 i 到 10 的时候就直接溜了，child0 和 shell 同时从 BLOCKED 变为 READY，让 child0/child1/shell 并发运行不是很优雅。不等待的话是这样
```
going to insert process 13 to ready queue.
going to schedule process 13 to run.
Parent print 9
going to insert process 14 to ready queue.
User exit with code:0.
going to insert process 0 to ready queue.
going to schedule process 14 to run.
Child0 print 9
going to insert process 15 to ready queue.
User exit with code:0.
going to schedule process 0 to run.
==========Command End============

User exit with code:0.
going to schedule process 15 to run.
Child1 print 9
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```
等的话是这样
```
going to insert process 15 to ready queue.
going to schedule process 15 to run.
Child1 print 8
going to insert process 13 to ready queue.
going to schedule process 13 to run.
Parent print 9
going to insert process 14 to ready queue.
going to schedule process 14 to run.
Child0 print 9
going to insert process 15 to ready queue.
going to schedule process 15 to run.
Child1 print 9
User exit with code:0.
going to insert process 14 to ready queue.
going to schedule process 14 to run.
User exit with code:0.
going to insert process 13 to ready queue.
going to schedule process 13 to run.
User exit with code:0.
going to insert process 0 to ready queue.
going to schedule process 0 to run.
==========Command End============

User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```
### lab3_challege3
这一关卡了我好久，典中典之 challenge 互相打架。
1. 子进程的堆在真正被写入数据的时候，相关的虚拟地址会发生 COW，在 pke 中的表现是触发 page fault 让你去处理。由于这个虚拟地址`stval`属于堆段而不属于栈段，lab2_challenge1 的代码会直接杀死比赛。这个问题很好解决，考虑一下如何兼容。
2. 有一个我自己作出来的问题，在 lab4_challenge2 重载执行的时候，我是把当前进程的堆段给 free 掉了，再装入新的程序内容。其实也可以不这么做，但是做了的话可以省一点物理空间。但是在 challengeX 中，app_cow 是 app_shell 的子进程，根据 lab3_challenge3 的 COW 要求，app_cow 和 app_shell 的堆段对应相同的物理页面，由于 app_shell 还要使用这个堆，所以 app_cow 只能 unmap，不能 free，问题解决。同理，如果你实现了数据段的 COW，然后在重载执行的时候还把进程的数据段所占物理页面 free 掉了，那么在 challengeX 中需要改 free 为 unmap。
> 一个仍有进程在使用的物理页`va`，如果 free 掉了，那么它里面的内容就可能会被改变，从而导致`((list_node*)va)->next`变成奇怪的物理地址。可能引发的问题是`epc`乱跳，用户程序的执行逻辑莫名其妙，无法解释。

## Lab4 Tips
### lab4_challenge1
这一关也卡了我很久，注意函数`strtok`，这是一个实现得非常糟糕的函数。如果对于多个字符串的拆分是穿插进行的，那么就会出现错误。
举个例子，有两个字符串：
```c
char s1[32] = "1!2!3";
char s2[32] = "a?b?c?d";
```
希望打印在控制台的是：
```
1
a b c d
2
a b c d
3
a b c d
```
看看下面的实现逻辑：
```c
void print_s2(){
	char s2_copy[32];
	strcpy(s2_copy,s2);
	char *token = strtok(s2_copy,"?");
	while(token){
		printu("%s ",token);
		token = strtok(NULL,"?");
	}
}

int main(){
	char s1_copy[32];
	strcpy(s1_copy,s1);
	char *token = strtok(s1_copy,"!"); // token = "1"
	while(token){
		printu("%s\n",token);
		print_s2();
		token = strtok(NULL,"!"); // 第一次执行到这里，token 是什么？
	}
	return 0;
}
```
上面的`main`函数包装隐藏得很好，粗略看一下没有什么问题。但是仔细观察`strtok`的实现，会发现在`main`内第一次执行到第 7 行`token`为`NULL`而不是预期的`"2"`。
challengeX 在加上 lab4_challenge1 的内容后就有这个问题，首先`app_shell`里面调用了`strtok`，而在执行`app_relativepath`的时候也**有可能使用**`strtok`。那么，`app_shell`接下来再`strtok`就会出错了。
### lab4_challenge2
这个其实也包含在 challengeX 里面了，`app_shell`的子进程相当于挑战实验的`app_exec`，而`app_ls`则是与 lab4_challenge3 共有的，所以也不用专门去做。
### lab4_challenge3
早就包含在 challengeX 里面了。

## Other Tips
### 虚拟地址为负数？
在做加分实验的过程中，有时候会出现用户程序全局变量虚拟地址为负数的情况。这种情况下，你可以使用下面的指令查看一下反汇编。正常情况下（所有的基础实验和挑战实验），编译器是**不会用**全局指针寄存器`gp`的。如果你用 objdump 看到了`gp`，那问题就找到了。
```bash
riscv64-unknown-elf-objdump -d ./hostfs_root/bin/app_shell
```
问题是什么？问题是编译器偶尔会抽风，使用`gp`寄存器，而我们简陋的 pke 实验显然没有考虑到编译器的这一行为。不信你可以在`app_shell.c`的 main 函数里面查看一下`gp`寄存器的值，绝对是 0。我们需要手动设置一下`gp`的内容，做法是直接在 main 函数开头加上这一句：
```c
asm volatile("la gp, __global_pointer$ \n\t");
```
问题解决。这里的符号`__global_pointer$`就是给寄存器`gp`进行初始化的，你也可以这样看这个符号的值：
```bash
riscv64-unknown-elf-readelf -s ./hostfs_root/bin/app_shell | grep -F "__global_pointer$"
```
结果示例：
```
25: 00000000000127ba     0 NOTYPE  GLOBAL DEFAULT  ABS __global_pointer$
```
> 然而，上面这些方法有时候未必能给 gp 寄存器赋值。实在是太难蹦了你科的 os 实验。

---

- 2 页个人简历
- 4 页申请表
- 2 页成绩单
- 1 页成绩证明
- 获奖证明 ：
	- 1 CSP
	- 3 蓝桥
	- 1 美赛
	- 1 数竞
	- 1 天梯赛
	- 1 自强
	- 2 三好
	- 1 国奖
- total: 20

---
# 保研面谈
## 个人介绍

## 英语问答（外语口语和听力）

## 综合问答（专业基础能力；科研综合能力和发展潜力；综合能力）

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE2MDYxNzEzNTIsMTM2OTk2NTIyNiwxMT
k1NzgwMzE1LC0xNTYxNTYwMzM0LDEyODU0MDAxMTUsMTg5MTAy
MDY0NiwtMjY5ODAyNjQ0XX0=
-->