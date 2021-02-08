# cgroup

cgroup はプロセスをグループ化し、そのグループに属するプロセスに対してリソースの制限を行う仕組みです。  
cgroup は cgroupfs と呼ばれるファイルシステムを通して操作します。多くは `/sys/fs/cgroup` にマウントされています。

制限できるリソースのことをサブシステムと呼び、CPU のコア数やメモリ使用量、プロセス数などを制限できます。  
サブシステムについては man をご参照ください[^1]。

## CPU の利用を制限する

無限ループのプロセスを作成し、そのプロセスに対して CPU 利用率を100%にしてみます。  
cgroup 管理下にない状態で `top` で確認すると 100% 利用していることが確認できます。

```sh
$ while true; do : ; done &
[1] 16541

$ top

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  16541 ubuntu    20   0   10252   1960      0 R 100.0   0.0   0:18.27 bash
```

このプロセスを cgroup 管理下に置きます。

```sh
$ sudo mkdir /sys/fs/cgroup/cpu/test
$ echo 16541 | sudo tee -a /sys/fs/cgroup/cpu/test/tasks
```

続いてCPU利用率を50%にするように調整します。すると 50% 前後に落ち着くことが確認できます。

```sh
$ echo 50000 | sudo tee -a /sys/fs/cgroup/cpu/test/cpu.cfs_quota_us
50000
$ top
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  16541 ubuntu    20   0   10252   1960      0 R  49.8   0.0   9:48.24 bash
```

## メモリ使用量を制限する

続いてメモリ使用量も200MBに制限してみます。

```sh
$ stress --vm 1 --vm-bytes 500M
stress: info: [17223] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd

$ top
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  17273 root      20   0  515860 291760    208 R 100.0   7.2   0:23.89 stress
```

プロセスを cgroup 管理下に置きます。

```sh
$ sudo mkdir /sys/fs/cgroup/memory/test
$ echo 17272 | sudo tee -a /sys/fs/cgroup/memory/test/tasks
17273
```

メモリ使用量を 200MB に制限します。

```sh
ubuntu@docker:~$ echo 200M | sudo tee -a /sys/fs/cgroup/memory/test/memory.limit_in_bytes
200M
```

すると `stress` が OOM Killer によって kill されます。

```sh
stress: FAIL: [17272] (415) <-- worker 17273 got signal 9
stress: WARN: [17272] (417) now reaping child worker processes
stress: FAIL: [17272] (451) failed run completed in 55s

$ dmesg
[23834.514678] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=/,mems_allowed=0,oom_memcg=/test,task_memcg=/test,task=stress,pid=17273,uid=0
[23834.514683] Memory cgroup out of memory: Killed process 17273 (stress) total-vm:515860kB, anon-rss:204432kB, file-rss:336kB, shmem-rss:0kB, UID:0 pgtables:444kB oom_score_adj:0
[23834.519784] oom_reaper: reaped process 17273 (stress), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
```

# docker で cgroup を使う

docker では `run` サブコマンドに対して `-c` フラグや `-m` フラグを指定することで制限できます。詳しくは `docker run --help` で確認してください。

```sh
$ docker run --rm -it -m 100m --memory-swappiness=0 ubuntu:20.04 bash
root@36570f483e85:/# stress --vm 2 --vm-bytes 100M
stress: info: [255] dispatching hogs: 0 cpu, 0 io, 2 vm, 0 hdd
stress: FAIL: [255] (415) <-- worker 257 got signal 9
stress: WARN: [255] (417) now reaping child worker processes
stress: FAIL: [255] (415) <-- worker 256 got signal 9
stress: WARN: [255] (417) now reaping child worker processes
stress: FAIL: [255] (451) failed run completed in 0s

$ dmesg
[24555.596613] Memory cgroup out of memory: Killed process 17691 (stress) total-vm:106260kB, anon-rss:45456kB, file-rss:320kB, shmem-rss:0kB, UID:0 pgtables:136kB oom_score_adj:0
[24555.596719] oom_reaper: reaped process 17692 (stress), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
[24555.600384] oom_reaper: reaped process 17691 (stress), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB

$ cat /sys/fs/cgroup/memory/docker/36570f483e8510887b7be178fba3830f1088aa694131c89a18043ef3db220658/memory.limit_in_bytes
104857600
```

---

[^1]: https://man7.org/linux/man-pages/man7/cgroups.7.html
