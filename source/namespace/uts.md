# UTS Namespace

UTS Namespace はホスト名の分離に利用されます。  
`uname(2)` や `gethostname(2)` を使用したときに Namespace 内で設定された値を取得することができます。  

`unshare(1)` の引数に `--uts` フラグを用いることで UTS Namespace を作成できます。  
`hostname(1)` で別のホスト名に変更してもホスト側には影響がないことが確認できます。

```sh
ubuntu@docker:~$ sudo unshare --uts bash
root@docker:/home/ubuntu# hostname test
root@docker:/home/ubuntu# hostname
test

# ホスト側
ubuntu@docker:~$ hostname
docker
```

