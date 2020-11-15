# chroot と pivot_root

Mount namespace では、名前空間ごとにマウントポイントを利用できることが確認できました。  
しかしコンテナではルートディレクトリ `/` 配下を全て別のファイルシステムにしなければ、ホスト側のファイルを操作できてしまいます。  
これを実現するために `chroot` と `pivot_root` が利用されます。

どちらもルートディレクトリを別のディストリに置き換えることができるシステムコールですが、挙動が全く異なります。

## chroot

`chroot` は現在のプロセスとその子プロセスのルートディレクトリを変更するシステムコールです。  
例えば次のように Alpine Linux の rootfs を用意し、そのディレクトリに chroot することで、ルートディレクトリが置き換えられたように見えます。

```sh
ubuntu@docker:~/$ mkdir alpine
ubuntu@docker:~/$ cd alpine
ubuntu@docker:~/alpine$ wget http://dl-cdn.alpinelinux.org/alpine/v3.12/releases/x86_64/alpine-minirootfs-3.12.1-x86_64.tar.gz
ubuntu@docker:~/alpine$ tar xzf alpine-minirootfs-3.12.1-x86_64.tar.gz
ubuntu@docker:~/alpine$ rm alpine-minirootfs-3.12.1-x86_64.tar.gz
ubuntu@docker:~/alpine$ cd ..

ubuntu@docker:~$ sudo chroot alpine sh
/ # ls
bin    etc    lib    mnt    proc   run    srv    tmp    var
dev    home   media  opt    root   sbin   sys    usr
/ # cat /etc/alpine-release
3.12.1
```

### chroot の問題点

chroot はプロセスが `CAP_SYS_CHROOT` Capability を持っている場合に、脱獄(chroot 環境から元の環境に移動できる)が可能です。  
これは chroot がカレントディレクトリを変更しないことに起因している仕様です。

プロセスのタスク構造体には、ルートディレクトリ情報を持つ `fs->root` とカレントディレクトリ情報を持つ `fs->pwd` があります。  
`chroot /path/to/debian` すると `fs->root` は `/path/to/debian` になります。  
さらにその chroot 環境下で `chroot test` すると `fs->root` は `/path/to/debian/test` になるのですが、 `fs->pwd` は `/path/to/debian` のままとなり、 root が pwd の子になっている構造になてしまいます。  
`cd ..` すると `fs->root` かどうかチェックが走りますが、このケースだと `fs->root` にマッチすることはないため、最終的に本来の root にたどり着き脱獄することができるという仕組みです。

これをコードにすると次のようになります。

```sh
$ cat jailbreak.c
#include <stdio.h>
#include <sys/stat.h>
#include <sys/types.h>

void main()
{
  mkdir("test", 0);
  chroot("test");
  chroot("../../../../../../../../../../");
  execv("/bin/bash");
}

$ gcc jailbreak.c
$ mv a.out debian/
$ sudo chroot debian bash
# ./a.out
/home/ubuntu/debian# cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04.1 LTS"
```

このように chroot できる権限を持っていると脱獄ができてしまいます。  
そこで脱獄を防ぐために pivot_root というシステムコールがあります。

## pivot_root

chroot はルートディレクトリを変更するものでしたが、pivot_root はプロセスのルートファイルシステムを入れ替えるものです。  
つまり、現在のプロセスのルートファイルシステムを別の場所にマウントし、新しいルートファイルシステムを `/` にマウントすることができます。  
全く別のものにすり替えてしまうものなので、脱獄のしようがありません。また、古いルートファイルシステムを unmount することも可能です。  

ただし、pivot_root をするには次の条件を満たす必要があります。[^1]

* 新しいファイルシステム(new_root)と元のファイルシステム(put_old)は現在のルートファイルシステムと同じマウントポイントにあってはいけない
* put_old は new_root の配下になければならない
* 他のファイルシステムを put_old にマウントできない

上記を満たすために bind mount を利用します。bind mount は指定したディレクトリを別の場所にそのままマウントします。インターフェイスとしては `ln` コマンドに似ていますが、一つのマウントポイントとして機能するため、pivot_root の条件を満たすことができます。

もうひとつ注意点として、pivot_root はマウントポイントを操作してルートディレクトリが変更されてしまうため、Mount Namespace を利用して実行することになります。  
では rootfs を差し替えたコンテナもどきを作ってみます。

```sh
ubuntu@docker:~$ sudo unshare --uts --pid --fork --mount sh -c \
  "mount --bind $NEW_ROOT $NEW_ROOT && \ # bind mount
  mount -t proc proc $NEW_ROOT/proc && \ # procfs をマウント
  pivot_root $NEW_ROOT $NEW_ROOT/.put_old && \ # pivot_root で差し替え
  umount -l /.put_old && \ # 元のルートファイルシステムを umount
  cd / && \
  exec /bin/sh"

/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    6 root      0:00 ps aux
/ # ls /etc
alpine-release  hosts           modules-load.d  periodic        shells
apk             init.d          motd            profile         ssl
conf.d          inittab         mtab            profile.d       sysctl.conf
crontabs        issue           network         protocols       sysctl.d
fstab           logrotate.d     opt             securetty       udhcpd.conf
group           modprobe.d      os-release      services
hostname        modules         passwd          shadow
```

---

[^1]: その他の条件など、詳しくは man https://man7.org/linux/man-pages/man2/pivot_root.2.html を参照ください
