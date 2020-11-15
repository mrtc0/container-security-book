# 1. コンテナの基礎技術

Linux コンテナはホストと分離するために様々な Linux カーネルの仕組みを利用しています。  
攻撃が生じる特定の領域に複数の保護レイヤを導入し、各レイヤは同種の攻撃に対して脆弱にならないような多層防御の仕組みが取られています。  
本章ではその保護の仕組みについて取り上げます。

## [Namespace](./namespace/README.md)

Linux コンテナのリソース分離技術の要である Namespace について簡単に紹介します。

## [chroot と pivot_root](./namespace/chroot-and-pivot_root.md)

コンテナのファイルシステムを分離するために必要な2つの仕組み `chroot` と `pivot_root` について紹介します。  
また、chroot の問題点についても取り上げます。

## [Capability](./capability/README.md)

Linux において権限を柔軟に付与できる仕組みである Capability について紹介します。

## [seccomp](./seccomp/README.md)

呼び出されるシステムコールの制限を行う seccomp について紹介します。

## [AppArmor](./lsm/apparmor.md)

ubuntu / debian などで、コンテナの保護機能の一つとして利用されている Linux Security Module である AppArmor について紹介します。

## [cgroup](./cgroup/README.md)

プロセスのリソース使用量を制限する仕組みである cgroup について紹介します。

# 2. コンテナのセキュリティと攻撃例

Linux コンテナの攻撃方法やそのセキュリティについて取り上げます。

## [Breakout Container](./security/breakout-to-host.md)

コンテナからホストに脱出することは Breakout / Jailbreak / Escape と呼ばれます。Privileged コンテナなど、過剰な権限を与えた場合にホスト側に Breakout できてしまうことを攻撃例を交えて少空きします。

## [Sensitive file mount](./security/sensitive-file-mount.md)

特定のファイルをマウントした場合にホスト側に Breakout できる攻撃例を紹介します。

## [DoS](./security/DoS.md)

コンテナからホスト側に対する DoS 攻撃例を紹介します。

## [コンテナ実行権限を持つグループへユーザー追加することについて](./security/adding-a-user-to-group.md)

Docker や LXD を操作するために一般ユーザーをそれぞれの実行権限を持つグループに追加した場合の危険性について紹介します。

## [AppArmor Bypass](./security/apparmor-bypass.md)

AppArmor のバイパス方法について紹介します。

## [seccomp Bypass](./security/seccomp-bypass.md)

seccomp のバイパス方法について紹介します。

## [イメージレイヤへの機密情報の保存](./security/image/secrets-in-layer.md)

イメージビルド時に機密情報をイメージに含めると、生成されたイメージから抽出することができるケースがあります。  
イメージレイヤの仕組みと具体例について紹介します。

## [イメージスキャナ](./security/image/scanner.md)

Docker イメージに含まれるソフトウェアの脆弱性を検出するイメージスキャナと、スキャナ自体の脆弱性について紹介します。

# 3. Hardening Container

コンテナをより安全に実行するための方法について紹介します。

## [No New Privilegjes](hardening/no-new-privileges.md)

子プロセスが特権を獲得しないようにする No New Privileges について紹介します。

