---
title: '/dev/stderr vs >&2: a Deep Dive Into Linux STDIO'
layout: post
author: Sichong Peng
date: '2021-11-06'
slug: dev-stdout-vs-2-a-dive-into-linux-stdio
categories:
  - Linux
tags:
  - Linux
image:
  caption: ''
  focal_point: ''
---

I was assisting a bioinformatics workshop for FAANG group at Plant and Animal Genome (PAG) and I ran into an interesting question with an example provided by one of the instructors on the topic of Linux output redirecting. Take a look at this simplified example script `print_stderr.sh`:

```bash
#!/usr/bin/bash

echo "first line of stderr" > /dev/stderr
echo "second line of stderr" > /dev/stderr
echo "third line of stderr" > /dev/stderr

echo "A line of stdout"
```

Now if we redirect the stderr of this script to a file, instead of all three lines of stderr messages, we only see the last line!
```
$ ./print_stderr.sh 2> err.log
$ cat err.log
third line of stderr
```

Why is this happening??

Now remember, when we "redirect" output of a command to stderr, we typically do it like so:
```
command >&2
```
What if we modify the script according to this?

```bash
#!/usr/bin/bash

echo "first line of stderr" >&2
echo "second line of stderr" >&2
echo "third line of stderr" >&2

echo "A line of stdout"
```

```bash
$ ./print_stderr.sh 2> err.log
$ cat err.log
first line of stderr
second line of stderr
third line of stderr
```

Now it works as we intended it to! So what's the difference?

Notice that I quoted "redirect" above. Because what ">&" idiom really does is duplicating, not redirecting. And that is the key to understand this interesting behaviour here.

To understand the difference, we first need to go over the three special file descriptors and how Linux handles its STDIO.

## What is a file descriptor
When a process starts, the kernel indexes all the files it requires and associate them with what are known as "file descriptors". In Linux systems, the first three files descriptors (`0`, `1`, and `2`) are by default `STDIN`, `STDOUT`, and `STDERR`. 

You may have heard that on a linux system, "everything is a file". And that is also true with I/O. `STDIN`, `STDOUT`, and `STDERR` all point to files on a system. When a process runs, it reads data from `STDIN` and writes to `STDOUT` and `STDERR`. Whether that `STDOUT` then goes to a console, a file, or something else is handled by the kernel. This level of absraction makes Linux particularly powerful and portable.

## How does redirecting work?

So when we write a command like this in a bash shell:
```
command 2> err.log
```
We are essentially telling the shell to write `STDERR` to a file `err.log` instead of its default place. 

Now where would that default place be? 

In an interactive shell, all three STDIO streams by default point to a device file. Many shells, including Bash, implement symlink files `/dev/stdin`, `/dev/stdout`, and `/dev/stderr`. These links point to `/proc/$pid/fd/0`, `/proc/$pid/fd/1`, `/proc/$pid/fd/2`, respectively. 

If you take a look at these files, you can see that they all point to a device file:

```bash
$ ls -lah /proc/self/fd/[0,1,2]
lrwx------ 1 sichong sichong 64 Jan  9 12:40 /proc/self/fd/0 -> /dev/pts/2
lrwx------ 1 sichong sichong 64 Jan  9 12:40 /proc/self/fd/1 -> /dev/pts/2
lrwx------ 1 sichong sichong 64 Jan  9 12:40 /proc/self/fd/2 -> /dev/pts/2
```

Not to go into too much detail, `/dev/pts/2` is a pseudo-terminal created by your current interactive shell. It allows the shell to read your keyboard input, as well as to write to your screen. Try writing something to this device file:

```bash
$ echo "Hello" > /dev/pts/2
Hello
```
It immediately shows up on your screen! 

Now you would ask, shouldn't every process have its own STDIO streams? If they all end up to the same place, how does the kernel know where each stream go to? 

Precisely! Every process gets its own set of file descriptors (stroed in `/proc/$pid/fd` where `$pid` is the process id). Now you probably know that when we run a script in an interactive shell, the shell calls a child process to run the script. So where do the STDIO streams of the child process point to?

It turns out that by default, the child process inherits from the parent process its STDIO targets. We can verify this by running a script like this(`show_file_descriptors.sh`):

```bash
#!/usr/bin/bash
filan -s
```
(You may need to install this program with `sudo apt install socat`)

And you should see an output that looks like this:
```
$ ./show_file_descriptors.sh
    0 tty /dev/pts/2
    1 tty /dev/pts/2
    2 tty /dev/pts/2
```

This tells us that our file descriptors all point to `/dev/pts/2` (same device file as our current shell session!) which is connected to a tty device.

Now what happens if I redirect the `STDERR` of this file to a different file?

```
$ ./show_file_descriptors.sh 2> err.log
    0 tty /dev/pts/2
    1 tty /dev/pts/2
    2 file /home/sichong/err.log
```

Now it's different! Our file descriptor `2` now points to the file `err.log`! 

To summarize, an interactive shell by default have all three of its STDIO streams (`STDIN`, `STDOUT`, `STDERR`) point to a same device file in `/dev/pts` directory. All child processes spawned by this shell session by dafault inherit the same file descriptors. (Hence when we run a program in a shell session it can take input from keyboard and we see its output on a screen, unless handled differently by the program.) But when we redirect input or output of a program, the spawned child process gets new file descriptors that point to redirected files.

## So, what happened?

At this point, it is probably clear to you already why the script we used at the workshop didn't work as intended:
The child process running the script gets `err.log` as its `STDERR`. This means that the `/dev/stderr` link points to `/proc/self/fd/2` which points to `err.log` instead of `/dev/pts/2`! So now when you redirect command of `echo` to `/dev/stderr` in the script, you're actually redirecting it to a file `err.log`. And since we are using `>` instead of `>>` to truncate the file, we only get whatever gets put in last.

To test our hypothesis, we can modify our script to use `>>` instead of `>` after each `echo` command. If our theory stands, this should make our script behave!

`print_stderr.sh`:

```bash
#!/usr/bin/bash

echo "first line of stderr" >> /dev/stderr
echo "second line of stderr" >> /dev/stderr
echo "third line of stderr" >> /dev/stderr

echo "A line of stdout"
```

Now let's try it again:


```bash
$ ./print_stderr.sh 2> err.log
$ cat err.log
first line of stderr
second line of stderr
third line of stderr
```

Tada!

## But why is `>&` different then?

To understand this, we need to dive deeper into how Linux systems handle redirecting. We are gonna use a program called `strace` to follow the system and see how it handles file descriptor at each step.
Try a simple command:

```bash
$ strace -f -e trace=openat,write,dup2 sh -c 'ls > out'
```
We would get output like this:
```
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "out", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
dup2(3, 1)                              = 1
strace: Process 10444 attached
[pid 10444] openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
[pid 10444] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3
[pid 10444] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 10444] openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libpcre2-8.so.0", O_RDONLY|O_CLOEXEC) = 3
[pid 10444] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
[pid 10444] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
[pid 10444] openat(AT_FDCWD, "/proc/filesystems", O_RDONLY|O_CLOEXEC) = 3
[pid 10444] openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
[pid 10444] openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
[pid 10444] write(1, "anaconda3\nandroid-studio\nApps\nbi"..., 268) = 268
[pid 10444] +++ exited with 0 +++
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=10444, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
dup2(10, 1)                             = 1
+++ exited with 0 +++
```
Some important lines to notice:
```
openat(AT_FDCWD, "out", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
```
This line indicates that the kernel opens file `out` and assignes it a file descriptor `3`.
Notice the `O_TRUNC` keyword here. This is where truncating happens with `>` redirecting: the system tries to open the target file, and truncate it if not empty, before the command is even evaluated!

> On a side note, this is also why doing `sort data.txt > data.txt` could potentially get you fired :)

After this step, the file is ready for writing and will take wahtever comes to it in sequential order, meaning no more truncating or overwriting will happen! (Unless another redirecting to the same file happens.)

```
dup2(3, 1)
```
This line suggests that the kernel duplicates the value of file descriptor `3` (which points to `out`) to file descriptor `1` (`STDOUT`).

Now at this point the `STDOUT` points to `out` instead of `/dev/pts/2`!
```
strace: Process 10444 attached
```
This process means that a child process is called to run `ls`
```
write(1, "anaconda3\nandroid-studio\nApps\nbi"..., 268) = 268
```
This line means that the output of `ls` is being written to file descriptor `1`, which points to `out`.

So this is what happens under the hood when we tell our shell to redirect the output of `ls` to a file named `out`!

> What if instead of `>` we use `>>` to redirect output of `ls`?

Now let's make a script (`redirect.sh`):

```bash
#!/usr/bin/bash
echo "Hello,"
echo "World!"
```
And let's trace what happens when we redirect output of this script:

```bash
$ strace -f -e trace=openat,write,dup2 sh -c './redirect.sh > out'
```

You should see output similar to this:
```
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "out", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
dup2(3, 1)                              = 1
strace: Process 10970 attached
[pid 10970] openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
[pid 10970] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libtinfo.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 10970] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
[pid 10970] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 10970] openat(AT_FDCWD, "/dev/tty", O_RDWR|O_NONBLOCK) = 3
[pid 10970] openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
[pid 10970] openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache", O_RDONLY) = 3
[pid 10970] openat(AT_FDCWD, "./test", O_RDONLY) = 3
[pid 10970] dup2(3, 255)                = 255
[pid 10970] write(1, "Hello,\n", 7)     = 7
[pid 10970] write(1, "World!\n", 7)     = 7
[pid 10970] +++ exited with 0 +++
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=10970, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
dup2(10, 1)                             = 1
+++ exited with 0 +++
```
This is very similar to what we see before. Notice that the child process does not attempt to open `out` file again! That is why `./redirect.sh > out` will print
```
Hello,
World!
```
to file `out` instead of only the last line!

But now let's modify our script:

```bash
#!/usr/bin/bash
echo "Hello," > /dev/stderr
echo "World!" > /dev/stderr
```

And redirect output of this script to a file:

```bash
$ strace -f -e trace=openat,write,dup2 sh -c './redirect.sh 2> err'
```

Now you will see output like this:
```
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "err", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
dup2(3, 2)                              = 2
strace: Process 11106 attached
[pid 11106] openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
[pid 11106] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libtinfo.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 11106] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
[pid 11106] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 11106] openat(AT_FDCWD, "/dev/tty", O_RDWR|O_NONBLOCK) = 3
[pid 11106] openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
[pid 11106] openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache", O_RDONLY) = 3
[pid 11106] openat(AT_FDCWD, "./test", O_RDONLY) = 3
[pid 11106] dup2(3, 255)                = 255
[pid 11106] openat(AT_FDCWD, "/dev/stderr", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
[pid 11106] dup2(3, 1)                  = 1
[pid 11106] write(1, "Hello,\n", 7)     = 7
[pid 11106] dup2(10, 1)                 = 1
[pid 11106] openat(AT_FDCWD, "/dev/stderr", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
[pid 11106] dup2(3, 1)                  = 1
[pid 11106] write(1, "World!\n", 7)     = 7
[pid 11106] dup2(10, 1)                 = 1
[pid 11106] +++ exited with 0 +++
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=11106, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
dup2(10, 2)                             = 2
+++ exited with 0 +++
```
Notice that the same file `err` is opened and truncated three times here:
```
openat(AT_FDCWD, "err", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
```
First it is opened by parent process and assigned file desccriptor `3`

```
dup2(3, 2) 
```
Then in the parent process the value of file descriptor `3` is copied to `2` (`STDERR`). And then child process is called which inherits the parent's `STDERR` (points to `err` at this point).

```
[pid 11106] openat(AT_FDCWD, "/dev/stderr", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
[pid 11106] dup2(3, 1)                  = 1
[pid 11106] write(1, "Hello,\n", 7)     = 7
```

And then the child process attempts to open the file it's supposed to redirect to (`/dev/stderr`), truncates it, and assigns it a file descriptor `3` (separately stored from parent's descriptors). Then it copies the value of file descriptor `3` to `1` (`STDOUT`). This completes the redirecting following `echo` command (`> /dev/stderr`). And finally it writes output of `echo` command to its `STDOUT` (which now points to `err` file).
 
And the same process repeats with the second `echo` command, removing output in the `err` file by the first `echo` command.

This all makes sense, but why is `>&` different?

Well let's see what our system does when we use `>&`:

But now let's modify our script:

```bash
#!/usr/bin/bash
echo "Hello," >& 2
echo "World!" >& 2
```

And run it again like this:

```bash
$ strace -f -e trace=openat,write,dup2 sh -c './redirect.sh 2> err'
```

Now check out the output:
```
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "err", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
dup2(3, 2)                              = 2
strace: Process 11593 attached
[pid 11593] openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
[pid 11593] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libtinfo.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 11593] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
[pid 11593] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 11593] openat(AT_FDCWD, "/dev/tty", O_RDWR|O_NONBLOCK) = 3
[pid 11593] openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
[pid 11593] openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache", O_RDONLY) = 3
[pid 11593] openat(AT_FDCWD, "./test", O_RDONLY) = 3
[pid 11593] dup2(3, 255)                = 255
[pid 11593] dup2(2, 1)                  = 1
[pid 11593] write(1, "Hello,\n", 7)     = 7
[pid 11593] dup2(10, 1)                 = 1
[pid 11593] dup2(2, 1)                  = 1
[pid 11593] write(1, "World!\n", 7)     = 7
[pid 11593] dup2(10, 1)                 = 1
[pid 11593] +++ exited with 0 +++
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=11593, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
dup2(10, 2)                             = 2
+++ exited with 0 +++
```

Notice this time, our file `err` is only opened and truncated once, before the child process starts! The child process simply copies value of file descriptor `2` to `1`, which redirects `STDOUT` to `STDERR` and immediately writes output of `echo` to `STDOUT` (points to `err` file at this point). Since there is no additional `openat()` call, the `err` file does not get truncated between two `echo` calls!

Intuitively, we can postulate that, when given a file path to redirect to, Linux will always attempt to open the file and assign a file descriptor to it, truncating the file in the process. But when redirecting with `>&`, Linux simply copies file descriptor, without attempting to open again (a valid file descriptor should indicate that the file is open and ready for I/O).

To verify this speculation, let's try two commands and compare them:

```bash
$ strace -f -e trace=openat,write,dup2 sh -c 'ls 1>&2'

openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
dup2(2, 1)                              = 1
strace: Process 12024 attached
[pid 12024] openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
[pid 12024] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3
[pid 12024] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 12024] openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libpcre2-8.so.0", O_RDONLY|O_CLOEXEC) = 3
[pid 12024] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
[pid 12024] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
[pid 12024] openat(AT_FDCWD, "/proc/filesystems", O_RDONLY|O_CLOEXEC) = 3
[pid 12024] openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
[pid 12024] openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
[pid 12024] write(1, "anaconda3\tbin\t Documents  exampl"..., 119anaconda3	bin	 Documents  examples.desktop  learn_linux  NASA_APOD  output	R	      snap	 token.pickle  WorkGoogleDrive
) = 119
[pid 12024] write(1, "android-studio\tDesktop  Download"..., 120android-studio	Desktop  Downloads  go		      local	   OneDrive   Pictures	restore.spst  Templates  Videos        Zotero
) = 120
[pid 12024] write(1, "Apps\t\tdlang\t err\t    guix-instal"..., 89Apps		dlang	 err	    guix-install.sh   Music	   out	      Public	scripts       test	 vim
) = 89
[pid 12024] +++ exited with 0 +++
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=12024, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
dup2(10, 1)                             = 1
+++ exited with 0 +++
```


```bash
$ strace -f -e trace=openat,write,dup2 sh -c 'ls >/dev/stderr'

openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/dev/stderr", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
dup2(3, 1)                              = 1
strace: Process 12080 attached
[pid 12080] openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
[pid 12080] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3
[pid 12080] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
[pid 12080] openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libpcre2-8.so.0", O_RDONLY|O_CLOEXEC) = 3
[pid 12080] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
[pid 12080] openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
[pid 12080] openat(AT_FDCWD, "/proc/filesystems", O_RDONLY|O_CLOEXEC) = 3
[pid 12080] openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
[pid 12080] openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
[pid 12080] write(1, "anaconda3\tbin\t Documents  exampl"..., 119anaconda3	bin	 Documents  examples.desktop  learn_linux  NASA_APOD  output	R	      snap	 token.pickle  WorkGoogleDrive
) = 119
[pid 12080] write(1, "android-studio\tDesktop  Download"..., 120android-studio	Desktop  Downloads  go		      local	   OneDrive   Pictures	restore.spst  Templates  Videos        Zotero
) = 120
[pid 12080] write(1, "Apps\t\tdlang\t err\t    guix-instal"..., 89Apps		dlang	 err	    guix-install.sh   Music	   out	      Public	scripts       test	 vim
) = 89
[pid 12080] +++ exited with 0 +++
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=12080, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
dup2(10, 1)                             = 1
+++ exited with 0 +++
```

Exactly what we predicted! We see that when redirected with `>&2`, the function `openat(AT_FDCWD, "/dev/stderr", O_WRONLY|O_CREAT|O_TRUNC, 0666)` is not called, while when redirected with `>/dev/stderr`, it does get called.

## To summarize...
You should pretty much always use `>&` instead of `/dev/stderr` or `/dev/stdout` when manipulating STDIO streams.