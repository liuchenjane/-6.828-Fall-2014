# 操作接口： operating system interface <br>
---

interface features: simple and narrow; <br>
接口：系统调用的集合 <br>
进程（process）:each running program has memory containing instructions, data, and a stack. <br>
shell: an ordinary program ,从用户那读程序再运行他们，是Unix-like系统最开始的接口。<br>

## 进程和内存(process and memory)

系统调用：` fork()`: 创建子进程，和父进程有相同的内存目录（memory contents)，fork返回子进程和父进程的pid；
子进程的pid==0，父进程内会返回子进程

eg:<br>
```c
int pid=fork();
if(pid>0){  //父进程
   printf("parent: child=%d\n",pid);
   pid=wait();   //wait():等待一个子进程退出，返回该子进程的pid
   printf(child %d is done\n",pid);
}else if(pid==0){
   printf("child: exiting\n");
   exit(); //exit():终止现在的进程,并释放资源
}else{
   printf("fork error\n");
}
```

输出：<br>
```
parent: child=1234
child: exiting
parent: child 1234 is done
```
也可能是：<br>
```
child: exiting
parent: child=1234
parent: child 1234 is done
```

因为这取决于the父进程还是子进程的printf先被调用<br>

**注意：父进程和子进程与不同的memery和register，因此一个变量的改变不会影响另外的一个**<br>

---

## 系统调用：exec(filename, *argv)————load a file and execute it <br>

从文件系统中load一个新文件，该文件具有ELF结构，并把它替换到调用进程内存，成功后，从ELF指定的入口点考试执行
eg:<br>
```c
char *argv[3];
argv[0]="echo";
argv[1]="hello";
argv[2]=0;
exec("/bin/echo",argv);
printf("exec error\n");
```
xv6系统通过上述的系统调用来运行用户的程序，以下是xv6 code中关于shell结构的部分：<br>
```c
8500 int
8501 main(void)
8502 {
8503 static char buf[100];
8504 int fd;
8505
8506 // Assumes three file descriptors open.
8507 while((fd = open("console", O_RDWR)) >= 0){
8508 if(fd >= 3){
8509 close(fd);
8510 break;
8511 }
8512
8513
8514 // Read and run input commands.
8515 while(getcmd(buf, sizeof(buf)) >= 0){//在循环中用getcmd读取命令
8516 if(buf[0] == ’c’ && buf[1] == ’d’ && buf[2] == ’ ’){
8517 // Clumsy but will have to do for now.
8518 // Chdir has no effect on the parent if run in the child.
8519 buf[strlen(buf)−1] = 0; // chop \n
8520 if(chdir(buf+3) < 0)
8521 printf(2, "cannot cd %s\n", buf+3);
8522 continue;
8523 }
8524 if(fork1() == 0)
8525 runcmd(parsecmd(buf));//runcmd运行实际的命令，调用exec 
8526 wait();
8527 }
8528 exit();
8529 }
```

读取命令后，调用fork创建一个shell进程的副本。父shell调用wait,当子进程在运行命令时<br>
fork和exec分开，而不放在一个函数调用，设计好的原因。<br>
xv6分配用户空间内存：fork为子进程分配空间，是父进程的copy；<br>
                    exec分配足够的空间以支持可运行文件(to hold the executable file)<br>
一个进程在运行是需要更多空间时，系统调用sbrk(n):数据空间增大n bytes.<br>
sbrk(n): grow process's memory by n bytes

---

## I/O and File descriptors(文件描述符)<br>
文件描述符是一个小的整数，它有内核管理的结构，可供进程读取和写入。<br>
file descriptor分配：the lowerest-numbered unused descriptor of the current process.<br>
the file descriptor interface abstracts away the differences between files, pipes, and devices, making them all look like streams of bytes.<br>
标准输入（standard input): file descriptor 0<br>
标准输出（standard input): file descriptor 1<br>
标准错误（standard error): file descriptor 2<br>

`read(fd,buf,n); `//从file descriptor fd中最多读取n字节，把他们拷贝进buf,返回拷贝的数量. <br>
每一个file descriptor都有相应的偏移量，一旦读了n字节，偏移量就会前进n字节。当没有字节可以读时，返回0，表明已到文件结尾。<br>
`write(fd,buf,n)`:从buf中取n字节写入file descriptor 中，返回写的字节数。<br>

下面是类似cat命令的本质片段：将标准输入的数据复制到标准输出
```c
char buf[512];
int n;

for(;;){
  n=read(0,buf,sizeof buf);
  if(n==0)
   break;
  if(n<0){
   fprintf(2,"read error\n");
   exit();
  }
  if(write(1,buf,n)!=n){
   fprintf("write error\n");
   exit();
  }
}
close(fd):释放打开文件fd,供以后的pipe,open,dup等使用。
```
<br>
file descriptor 和fork的交互使用，能使I/O的重定向变得简单。<br>
fork将父进程的file descriptor table及响应的内存复制，让子进程开始时和父进程一样。<br>
系统调用exec替换调用进程内存，但保留它的file table。这种行为允许shell通过fork实现I/O重定向，重新打开选定的file descriptor 然后运行新程序。<br>

下面code是一个简化版的shell怎么实现命令：cat < input.txt<br>
```c
char *argv[2];

argv[0]="cat";
argv[1]=0;
if(fork()==0){
 close(0);
 open("input.txt",O_RDONLY);
 exec("cat",argv);
}
```
首先，子进程关闭file descriptor 0,以确保当打开input.txt时，目前可用的最小的file descriptor 是0，这样运行程序时，标准输入指向input.txt。<br>
由此可见，将**fork与exec分开有什么好处呢？**
**这种机制允许子进程在实际运行目标程序前，可以对子进程进行修改，如这里的close和open就是修改。如果fork和exec是一个整体，这里就不可修改了。**
This separation allows the shell to fix up the child process before the child runs the intended program.

虽然fork复制了flie descriptor table,但他们共享 file offset.
eg:<br>
```c
if(fork()==0){
  write(1,"hello ",6);
  exit();
}else {
  wait(); //这里wait，可以保证child跑完后，在跑parent
  write(1,"world\n",6);
}
```
标准输出：hello world

系统调用dup(fd): 复制fd，使得返回的新的file descriptor和原来的指向同一个I/O object。
下面也是讲一个 hello world写入一个文件
```c
fd=dup(1);
write(1,"hello ",6);
write(fd,"world\n",6);
```
只有通过fork和dup得到的file descriptor才和原来的fd共享offset。
命令应用：
`ls existing-file non-existing-file > tmp1 2>&1`
2>&1：告诉shell, file descriptor 2 是 file descriptor 1 的的副本，已经存在文件的名字和不存在文件名字的错误信息都会显示在tmp1 <br>

---

## 管道(pipe)<br>
管道是一个小的内核缓存(kernel buffer),和file descriptor（用来读的）组成一对，pipe 用来写
pipe是进程通信一种方式.
系统调用：pipe(p) 创建一个管道，并在p中返回fd,分别记录读和写的file descriptor,p[0]是读管道，p[1]是写管道。

以下运行wc命令，并将标准输入和读管道关联。
```c
int p[2];
char *argv[2];

argv[0]= "wc";
argv[1]= 0;

pipe(p);
if(fork()==0){
  close(0);
  dup(p[0]);
  close(p[0]);
  close(p[1]);
  exec("/bin/wc",argv);
}else{
  write(p[1],"hello world\n", 12);
  close(p[0]);
  close(p[1]);
}
```
fork之后，parent和child都有指向这个pipe的file descriptor,其中的child将读管道复制到标准输出，然后关闭p中的file descriptor，运行wc。<br>
当wc从它的标准输入中读时，也是从pipe中读。当没有合适的待读数据时，读管道会一直等待新的数据写入或者所有指向写管道的file descriptor都关闭。因此，在运行wc之前，一定要将写管道关闭，否则会导致读入一直无法到达结尾。

xv6 shell实现的管道：<br>
eg: `grep fork sh.c | wc -l`
子进程创建一个pipe，将pipe的左右两端相连，它运行的方式与下面代码体现的类似，分别为管道的左右两边调用runcmd,也要等待左边和右边都完成，调用两次wait.<br>
```c
8450 case PIPE:
8451 pcmd = (struct pipecmd*)cmd;
8452 if(pipe(p) < 0)
8453 panic("pipe");
8454 if(fork1() == 0){
8455 close(1);
8456 dup(p[1]);
8457 close(p[0]);
8458 close(p[1]);
8459 runcmd(pcmd−>left);
8460 }
8461 if(fork1() == 0){
8462 close(0);
8463 dup(p[0]);
8464 close(p[0]);
8465 close(p[1]);
8466 runcmd(pcmd−>right);
8467 }
8468 close(p[0]);
8469 close(p[1]);
8470 wait();
8471 wait();
8472 break;
```
pipe 和临时文件
echo hello world | wc
等价于 echo hello world >/tmp/xyz; wc </tmp/xyz
**两者区别：
pipe会自动清除(lean up);但使用文件重定向，shell必须谨慎删除；
pipe可以通过任意长的数据流，但使用文件重定向时必须保证磁盘上有足够的空闲空间存储数据；
pipe允许同步：两个进程可以通过一对管道来回发送信息 <br>

---

## 文件系统(File system)
xv6 把目录当做一种特殊的文件，目录以树的形势组织，从根目录/开始。
系统调用chdir(dirname):改变当前目录
下面的部分代码打开的目录相同：
```c
chdir("/a");
chdir("b");
open("c",O_RDONLY);
```
open("a/b/c",O_RDONLY);
第一个将当前目录改变到/a/b. 第二个不会引用(refer to)或者改变当前目录


系统调用 mkdir(dirname):创建一个新目录,with O_CREATE flag
mknod(dirname):创建一个新的设备文件(device file)
eg:
```c
mkdir("/dir");
fd=open("/dir/file",O_CREATE|O_WRONLY);
closed(fd);
mknod("/console",1,1);
```
mknod创建的文件在文件系统中，但文件没有目录，而是通过metadata将其标记为一个设备文件，并记录它的major和minor number.

系统调用fstat(fd):返回打开文件的信息，在数据结构stat中显示，stat结构如下：
```c
#define T_DIR 1 //Directory
#define T_FILE 2 //File
#define T_DEV 3 //Device

struct stat {
  short type; //Type of file
  int dev;    //File system's disk device
  unit ino;   //Inode number
  short nlink; //number of links to file
  uint size; //size of file in bytes
};
```
文件名不同于文件本身；索引节点(inode)-the same underlying file可以有不同的名字----link（链接）

系统调用link(f1,f2):为文件f1创建另一个别名f2，它们指向同一个索引节点；<br>
`unlink(filename)`:删除一个文件名，当nlink为0时，磁盘会释放该目录的空间 <br>
下面创建一个文件，名字是a，也是b <br>
```c
open("a",O_CREATE|O_WRONLY);
link("a","b");
```
每一个索引节点都是通过索引节点号(inode number)唯一标记的，即stat中的ino.<br>
如果将
`unlink("a");`
加到上述代码片的后面，那么仅可以通过b去访问inode。<br>
eg:<br>
```c
fd= open("/tmp/xyz",O_CREATE_O_RDWR);
unlink("/tmp/xyz");
```
创建一临时索引节点，一旦进程close(fd)或者exit,相应的inode就会被清除。
xv6对文件系统的操作命令一般都是用户级程序。
但有一个例外是cd，built into the shell.cd必须改变shell当前正在工作的目录。如果cd被当做一个普通的命令运行，那么shell就会fork一个子进程，子进程会运行cd，那么cd改变的就是子进程的工作目录，而它父进程的工作目录不会改变。
