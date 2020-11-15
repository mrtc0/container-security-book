# イメージスキャン

Docker イメージにはアプリケーションの動作に必要なソフトウェアが含まれており、それらに脆弱性が存在することがあります。  
コンテナに限らず、オンプレやVMでも同様ですが、それらの脆弱性を利用されて権限昇格されることもあります。そのため、イメージに含まれる脆弱性の把握とリスク管理を行う必要があります。

ここではイメージをスキャンして脆弱性を把握するためのツールを紹介します。

## trivy

[https://github.com/aquasecurity/trivy][1]

trivy はイメージを静的解析し、主要OSのパッケージに加えて bundler や npm などでインストールされているアプリケーションパッケージもスキャン対象に含めることができます。  
他のイメージスキャナと比較して誤検知の少なさなど、正確性も売りとなっています。

```sh
$ trivy ubuntu:latest

ubuntu:latest (ubuntu 20.04)
============================
Total: 20 (UNKNOWN: 0, LOW: 18, MEDIUM: 2, HIGH: 0, CRITICAL: 0)

+-------------+------------------+----------+------------------------+-------------------+--------------------------------+
|   LIBRARY   | VULNERABILITY ID | SEVERITY |   INSTALLED VERSION    |   FIXED VERSION   |             TITLE              |
+-------------+------------------+----------+------------------------+-------------------+--------------------------------+
| bash        | CVE-2019-18276   | LOW      | 5.0-6ubuntu1.1         |                   | bash: when effective UID is    |
|             |                  |          |                        |                   | not equal to its real UID      |
|             |                  |          |                        |                   | the...                         |
+-------------+------------------+          +------------------------+-------------------+--------------------------------+
| coreutils   | CVE-2016-2781    |          | 8.30-3ubuntu2          |                   | coreutils: Non-privileged      |
|             |                  |          |                        |                   | session can escape to the      |
|             |                  |          |                        |                   | parent session in chroot       |
+-------------+------------------+          +------------------------+-------------------+--------------------------------+
| gpgv        | CVE-2019-13050   |          | 2.2.19-3ubuntu2        |                   | GnuPG: interaction between the |
|             |                  |          |                        |                   | sks-keyserver code and GnuPG   |
|             |                  |          |                        |                   | allows for a Certificate...    |
+-------------+------------------+          +------------------------+-------------------+--------------------------------+
| libc-bin    | CVE-2016-10228   |          | 2.31-0ubuntu9.1        |                   | glibc: iconv program can       |
|             |                  |          |                        |                   | hang when invoked with the -c  |
|             |                  |          |                        |                   | option                         |
+             +------------------+          +                        +-------------------+--------------------------------+
|             | CVE-2020-6096    |          |                        |                   | glibc: signed comparison       |
|             |                  |          |                        |                   | vulnerability in the ARMv7     |
|             |                  |          |                        |                   | memcpy function                |
+-------------+------------------+          +                        +-------------------+--------------------------------+
...
```

## snyk

[snyk](snyk.io) はアプリケーションライブラリの脆弱性DBを持ち、検知するツールを提供しています。Docker のイメージスキャンに snyk が利用されるようになり、Docker 2.3.6.0 以降は `docker scan` コマンドだけで[イメージスキャンが利用できる][2]ようになっています。  

```sh
❯ docker scan ubuntu:latest

Testing ubuntu:latest...

✗ Low severity vulnerability found in tar
  Description: NULL Pointer Dereference
  Info: https://snyk.io/vuln/SNYK-UBUNTU2004-TAR-576242
  Introduced through: meta-common-packages@meta
  From: meta-common-packages@meta > tar@1.30+dfsg-7

✗ Low severity vulnerability found in systemd/libsystemd0
  Description: Improper Input Validation
  Info: https://snyk.io/vuln/SNYK-UBUNTU2004-SYSTEMD-576079
  Introduced through: systemd/libsystemd0@245.4-4ubuntu3.2, apt@2.0.2ubuntu0.1, procps/libprocps8@2:3.3.16-1ubuntu2, util-linux/bsdutils@1:2.34-0.1ubuntu9.1, util-linux/mount@2.34-0.1ubuntu9.1, systemd/libudev1@245.4-4ubuntu3.2
  From: systemd/libsystemd0@245.4-4ubuntu3.2
  From: apt@2.0.2ubuntu0.1 > systemd/libsystemd0@245.4-4ubuntu3.2
  From: procps/libprocps8@2:3.3.16-1ubuntu2 > systemd/libsystemd0@245.4-4ubuntu3.2
  and 6 more...
...
```

## Anchore

[https://github.com/anchore/anchore-engine][3]

Anchore はイメージの脆弱性を集中管理する機能を持ちます。REST API を通して利用できるためプログラマブルであることが特徴の一つです。

```sh
❯ anchore-cli image add docker.io/library/debian:latest

Image Digest: sha256:60cb30babcd1740309903c37d3d408407d190cf73015aeddec9086ef3f393a5d
Parent Digest: sha256:8414aa82208bc4c2761dc149df67e25c6b8a9380e5d8c4e7b5c84ca2d04bb244
Analysis Status: not_analyzed
Image Type: docker
Analyzed At: None
Image ID: 1510e850178318cd2b654439b56266e7b6cbff36f95f343f662c708cd51d0610
Dockerfile Mode: None
Distro: None
Distro Version: None
Size: None
Architecture: None
Layer Count: None

Full Tag: docker.io/library/debian:latest
Tag Detected At: 2020-11-15T04:23:09Z

❯ anchore-cli image list
Full Tag                               Image Digest                                                                   Analysis Status
docker.io/library/debian:latest        sha256:60cb30babcd1740309903c37d3d408407d190cf73015aeddec9086ef3f393a5d        analyzed
docker.io/library/ubuntu:latest        sha256:1d7b639619bdca2d008eca2d5293e3c43ff84cbee597ff76de3b7a7de3e84956        analyzed

❯ anchore-cli image vuln docker.io/library/debian:latest os
Vulnerability ID        Package                            Severity          Fix         CVE Refs        Vulnerability URL                                                   Type        Feed Group        Package Path
CVE-2011-3389           libgnutls30-3.6.7-4+deb10u5        Medium            None                        https://security-tracker.debian.org/tracker/CVE-2011-3389           dpkg        debian:10         pkgdb
CVE-2005-2541           tar-1.30+dfsg-6                    Negligible        None                        https://security-tracker.debian.org/tracker/CVE-2005-2541           dpkg        debian:10         pkgdb
...
```

# イメージスキャナ自体の脆弱性

イメージスキャナはイメージを静的解析するものと内部で OS コマンドやパッケージマネージャーを実行する動的解析するものがあります。  
動的解析するものにOSコマンドの呼び出しに不備があると、不正なイメージファイルをスキャンさせることで、任意コード実行につなげることができます。

例えば Anchore 0.7 では次のように OS コマンドが呼び出され、そのバリデーションに不備があったため、任意コード実行につなげることができました。

* https://github.com/anchore/anchore-engine/issues/430

中には[脆弱性報告されたものの未修正のものもある][4]ため、利用ケースを考えて使用したり、イメージスキャナ自体もアップデートしていく必要があります。

[1]: https://github.com/aquasecurity/trivy/ "https://github.com/aquasecurity/trivy/"
[2]: https://docs.docker.com/engine/scan/ "https://docs.docker.com/engine/scan/"
[3]: https://github.com/anchore/anchore-engine "https://github.com/anchore/anchore-engine"
[4]: https://medium.com/@matuzg/testing-docker-cve-scanners-part-2-5-exploiting-cve-scanners-b37766f73005 "Testing docker CVE scanners. Part 2.5 — Exploiting CVE scanners"
