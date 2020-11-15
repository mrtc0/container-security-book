# イメージレイヤへの機密情報の保持

Docker イメージは OverlayFS のようにレイヤが存在し、ベースとなる OS レイヤに対してアプリケーションやライブラリを追加されたものになっています。

例えば次のような Dockerfile を用意しビルドします。

```sh
$ cat Dockerfile
FROM alpine:latest
RUN echo "secret" > /secret.txt
RUN rm /secret.txt

$ docker build -t test .
Sending build context to Docker daemon  10.61MB
Step 1/3 : FROM alpine:latest
 ---> d6e46aa2470d
Step 2/3 : RUN echo "secret" > /secret.txt
 ---> Running in 32f150d1804c
Removing intermediate container 32f150d1804c
 ---> 2cac5efedab4
Step 3/3 : RUN rm /secret.txt
 ---> Running in be0569fd1744
Removing intermediate container be0569fd1744
 ---> b29dd8898773
Successfully built b29dd8898773
Successfully tagged test:latest
```

この Dockerfile は3つのレイヤで構成されています。

* Layer 1 ... `FROM alpine:latest`
* Layer 2 ... `RUN echo "secret" > /secret.txt`
* Layer 3 ... `RUN rm /secret.txt`

より視覚的に確認するために [dive](https://github.com/wagoodman/dive) を使ってみると、確かに3つのレイヤであることを確認できます。

```
$ dive test
┃ ● Layers ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │ Current Layer Contents ├────────────────────
Cmp   Size  Command                            ├── bin
    5.6 MB  FROM b1c62b187dcd114               │   ├── arch → /bin/busybox
       7 B  echo "secret" > /secret.txt        │   ├── ash → /bin/busybox
       0 B  rm /secret.txt                     │   ├── base64 → /bin/busybox
                                               │   ├── bbconfig → /bin/busybox
│ Layer Details ├───────────────────────────── │   ├── busybox
                                               │   ├── cat → /bin/busybox
Tags:   (unavailable)                          │   ├── chgrp → /bin/busybox
Id:     b1c62b187dcd114a7252e45a4f03577549d822 │   ├── chmod → /bin/busybox
77149b5467c73eefaa956260bd                     │   ├── chown → /bin/busybox
Digest: sha256:ace0eda3e3be35a979cec764a3321b4 │   ├── conspy → /bin/busybox
c7d0b9e4bb3094d20d3ff6782961a8d54              │   ├── cp → /bin/busybox
Command:                                       │   ├── date → /bin/busybox
#(nop) ADD file:f17f65714f703db9012f00e5ec98d0 │   ├── dd → /bin/busybox
b2541ff6147c2633f7ab9ba659d0c507f4 in /        │   ├── df → /bin/busybox
                                               │   ├── dmesg → /bin/busybox
│ Image Details ├───────────────────────────── │   ├── dnsdomainname → /bin/busybox
                                               │   ├── dumpkmap → /bin/busybox
                                               │   ├── echo → /bin/busybox
Total Image size: 5.6 MB                       │   ├── ed → /bin/busybox
Potential wasted space: 7 B                    │   ├── egrep → /bin/busybox
Image efficiency score: 99 %                   │   ├── false → /bin/busybox
                                               │   ├── fatattr → /bin/busybox
```

イメージは各レイヤを保持しているため、特定のレイヤを抽出することができます。  
つまり、`rm /secret.txt` のように Dockerfile 内で機密情報を削除している場合でも、その機密情報を取り出すことが可能です。

```sh
$ docker save test > test.tar
$ mkdir test
$ cd test
~/test$ tar -xf ../test.tar
~/test$ ls
b1c62b187dcd114a7252e45a4f03577549d82277149b5467c73eefaa956260bd       b6433cc45f11a118c68ef34b9b3192f7c3514ee1a36c85d26df80122d058af4a  manifest.json
b29dd88987735c67e31b37de3c0c44abf656b42e6ce396defbfd967ba772e027.json  c8e83ef6a497050f640412539ecd335c4f7cf72808a75aea4ef1d0e04bb28156  repositories

$ cat b29*.json | jq
"history": [
    {
      "created": "2020-10-22T02:19:24.33416307Z",
      "created_by": "/bin/sh -c #(nop) ADD file:f17f65714f703db9012f00e5ec98d0b2541ff6147c2633f7ab9ba659d0c507f4 in / "
    },
    {
      "created": "2020-10-22T02:19:24.499382102Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/sh\"]",
      "empty_layer": true
    },
    {
      "created": "2020-11-02T09:05:38.780508124Z",
      "created_by": "/bin/sh -c echo \"secret\" > /secret.txt"
    },
    {
      "created": "2020-11-02T09:05:39.890868911Z",
      "created_by": "/bin/sh -c rm /secret.txt"
    }

$ cat manifest.json | jq
...
    "Layers": [
      "b1c62b187dcd114a7252e45a4f03577549d82277149b5467c73eefaa956260bd/layer.tar",
      "c8e83ef6a497050f640412539ecd335c4f7cf72808a75aea4ef1d0e04bb28156/layer.tar",
      "b6433cc45f11a118c68ef34b9b3192f7c3514ee1a36c85d26df80122d058af4a/layer.tar"
...

$ tar xf c8e83ef6a497050f640412539ecd335c4f7cf72808a75aea4ef1d0e04bb28156/layer.tar
$ cat secret.txt
secret
```

上記のようなケースを防ぐために、機密情報は環境変数で渡したりするようにしましょう。

# References

* [https://docs.docker.com/engine/swarm/secrets/](https://docs.docker.com/engine/swarm/secrets/)
* [https://docs.docker.com/develop/develop-images/build_enhancements/](https://docs.docker.com/develop/develop-images/build_enhancements/)
