# ランタイムセキュリティ

コンテナのセキュリティはホストとの Isolation に依存します。もしランタイムやカーネルに脆弱性があった場合、コンテナからホスト側に Breakeout できる要因になってしまいます。  

そこで、ランタイムを別のもの切り替えることでより強固にできるケースがあります。Docker はデフォルトで OCI ランタイムに runc を、CRI ランタイムに containerd を利用しますが、これらを gVisor や kata-containers などに切り替えることができます。

## gVisor

gVisor[^1] はコンテナのためのアプリケーションカーネルで、コンテナで発行されるシステムコールをトレースします。トレースされたシステムコールは gVisor によってフィルタ&置換されて実行されます。つまり、ホストカーネルのように振る舞うことで、ホストへの影響を小さくすることができています。  

一方で、gVisor が対応していないシステムコールは発行することができないため、例えば tcpdump などを実行することが 2020/11/15 現在できません。  
また、システムコールをトレースするということは、それだけオーバーヘッドが生じます。

## kata-containers

kata-containers[^2] はコンテナを VM のような分離レベルで動かすランタイムです。コンテナは QEMU / KVM / Firecraker 上で動作し、ホストとカーネルを共有しないため、非常に強固と言えます。  
このような Virtualized Containers への攻撃手法については black hat USA 2020 - Escaping Virtualized Containers[^3] に詳しく書かれています。

## Unikernel

TBD

---

[^1]: https://gvisor.dev/
[^2]: https://katacontainers.io/
[^3]: https://i.blackhat.com/USA-20/Thursday/us-20-Avrahami-Escaping-Virtualized-Containers.pdf
