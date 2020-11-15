# User Namespace

User Namespace は UID / GID を分離し、 Namespace 内で独立した UID / GID を持てるようになります。  
また、Namespace 内の UID / GID がそれぞれホスト側の UID / GID と mapping されるようになります。

例えば Namespace 内では UID 0 (root) であっても、ホスト側から見ると UID 1000 のユーザーであるようにできます。  
これにより仮にコンテナからホスト側にエスケープできても、権限は UID 1000 の一般ユーザーであるため、影響を小さくすることができます。

`unshare(1)` では `--user` フラグを用いることで User Namespace を作成できます。  

```sh
ubuntu@docker:~$ unshare --user bash
nobody@docker:~$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
nobody@docker:~$ echo $$
2275
```

現在、名前空間内では `nobody` ユーザーになっています。  
試しにホスト側の UID 1000 のユーザーと Namespace 内の UID 0 (root) のユーザーを紐付けてみます。
紐付けは対象のプロセスが持つ `uid_map` に値を書き込むことで機能します。

```sh
# ホスト側で操作
root@docker:/home/ubuntu# echo '0 1000 1' > /proc/2275/uid_map
```

名前空間内で確認すると UID 0 (root) になっていることが確認できます。

```sh
nobody@docker:~$ id
uid=0(root) gid=65534(nogroup) groups=65534(nogroup)
```

ファイルを作成すると名前空間内では root 所有に見えますが、実際には UID 1000 である `ubuntu` ユーザーの所有になっていることが確認できます。

```sh
nobody@docker:~$ touch test.txt
nobody@docker:~$ ls -al test.txt
-rw-rw-r-- 1 root nogroup 0 Nov 13 14:10 test.txt

# ホスト側で操作
ubuntu@docker:~$ ls -al test.txt
-rw-rw-r-- 1 ubuntu ubuntu 0 Nov 13 14:10 test.txt
```
