# seccomp

AppArmor と同様に seccomp プロファイルもコンテナ上で動作するアプリケーションのイベントをトレースしなければ生成することが困難です。  
発行されているシステムコールをトレースするには `strace` や eBPF が利用できますが、ここでも `docker-slim` を活用することができます。  

AppArmor のときと同様に実行すると `home/ubuntu/dist_linux/.docker-slim-state/images/9140108b62dc87d9bb278bb0d4fd6a3e44c2959646eb966b86531306faa81b09b/artifacts/ubuntu-seccomp.json` に seccomp プロファイルが生成されます。

生成された seccomp プロファイルを見ると、禁止されるシステムコールが明示的に記述されていることが確認できます。

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    {
      "names": [
        "getgid",
        "read",
        "setuid",
        "dup3",
        "getppid",
        ...
      ],
      "action": "SCMP_ACT_ALLOW",
      "includes": {},
      "excludes": {}
    }
  ]
}
```
