# 第 6.2 节 ZFS

## 使用建议

- 建议在 8G 及以上的内存机器上使用 ZFS。
- 为了提高机械硬盘随机读能力，可设置 `vfs.zfs.prefetch_disable=1`。
- 为了避免 ZFS 吃掉太多内存，可设置 `vfs.zfs.arc_max="XXX"`，例如：1024 M。
- 如果要复制某个文件系统，可以用 `zfs send/recv`，这样还能通过 ssh 跨网络传输。


以上部分来自网络，更多优化见 [ZFSTuningGuide](https://wiki.freebsd.org/ZFSTuningGuide)。

## ZFS 快照与还原

ZFS 快照类似于虚拟机快照。

- 创建快照

默认创建分区（Auto ZFS）如下：

```shell-session
root@ykla:/home/ykla #  zfs list
NAME                 USED  AVAIL     REFER  MOUNTPOINT
zroot               1.72G   440G       96K  /zroot
zroot/ROOT          1004M   440G       96K  none
zroot/ROOT/default  1004M   440G     1004M  /
zroot/tmp            104K   440G      104K  /tmp
zroot/usr            760M   440G       96K  /usr
zroot/usr/home       128K   440G      128K  /usr/home
zroot/usr/ports       96K   440G       96K  /usr/ports
zroot/usr/src        759M   440G      759M  /usr/src
zroot/var            628K   440G       96K  /var
zroot/var/audit       96K   440G       96K  /var/audit
zroot/var/crash       96K   440G       96K  /var/crash
zroot/var/log        148K   440G      148K  /var/log
zroot/var/mail        96K   440G       96K  /var/mail
zroot/var/tmp         96K   440G       96K  /var/tmp
```

快照 `zroot`（经测试，在上述默认分区下代表快照整个 ZFS 文件系统，`-r` 即递归创建快照，`test` 是随便起的名字）：

```shell-session
root@ykla:/home/ykla # zfs snapshot -r zroot@test
root@ykla:/home/ykla # zfs list -t snap
NAME                      USED  AVAIL     REFER  MOUNTPOINT
zroot@test                  0B      -       96K  -
zroot/ROOT@test             0B      -       96K  -
zroot/ROOT/default@test     0B      -     7.18G  -
zroot/tmp@test              0B      -      176K  -
zroot/usr@test              0B      -       96K  -
zroot/usr/home@test         0B      -     31.1M  -
zroot/usr/ports@test        0B      -     1.98G  -
zroot/var@test              0B      -       96K  -
zroot/var/log@test          0B      -      444K  -
root@ykla:/home/ykla # ls /usr/ports/
CHANGES          archivers/       emulators/       misc/            textproc/
CONTRIBUTING.md  astro/           finance/         multimedia/      ukrainian/
COPYRIGHT        audio/           french/          net-im/          vietnamese/
GIDs             base/            ftp/             net-mgmt/        www/
INDEX-13         benchmarks/      games/           net-p2p/         x11-clocks/
Keywords/        biology/         german/          net/             x11-drivers/
MOVED            cad/             graphics/        news/            x11-fm/
Makefile         chinese/         hebrew/          polish/          x11-fonts/
Mk/              comms/           hungarian/       ports-mgmt/      x11-servers/
README           converters/      irc/             portuguese/      x11-themes/
Templates/       databases/       japanese/        print/           x11-toolkits/
Tools/           deskutils/       java/            russian/         x11-wm/
UIDs             devel/           korean/          science/         x11/
UPDATING         distfiles/       lang/            security/
accessibility/   dns/             mail/            shells/
arabic/          editors/         math/            sysutils/
root@ykla:/home/ykla # rm /usr/ports/
```

- 还原快照

还原时不能递归还原快照，必须挨个还原（如果你有更好的方案请告诉我们,网络上有一些脚本可用）：

与虚拟机快照有所不同，在缺省情况下，`zfs rollback` 命令无法回滚到除最新快照以外的快照（[参考手册](https://docs.oracle.com/cd/E19253-01/819-7065/gbcxk/index.html)），除非使用`r`，但这会删除该快照创建后的所有快照。

```shell-session
root@ykla:/home/ykla # zfs rollback -r zroot@test
root@ykla:/home/ykla # zfs rollback -r zroot/ROOT@test
root@ykla:/home/ykla # zfs rollback -r zroot/ROOT/default@test
root@ykla:/home/ykla # zfs rollback -r zroot/tmp@test
root@ykla:/home/ykla # zfs rollback -r zroot/usr@test
root@ykla:/home/ykla # zfs rollback -r zroot/usr/home@test
root@ykla:/home/ykla # zfs rollback -r zroot/usr/ports@test
root@ykla:/home/ykla # zfs rollback -r zroot/var@test
root@ykla:/home/ykla # zfs rollback -r zroot/var/log@test
```

- 销毁快照

销毁快照（销毁的时候可以使用`r`递归销毁）：

```shell-session
root@ykla:/home/ykla # zfs destroy -r zroot@test
root@ykla:/home/ykla # zfs list -t snap
no datasets available
root@ykla:/home/ykla #
```

> `snapshot`在命令中可以缩写为`snap`。


## 启动环境

什么是启动环境？用 snapshot 和 rollback 结合，相当于在一条时间上线进行跳转。启动环境相当于一条时间线，复制一个启动环境相当于再造一条时间线(复制之后两个启动环境互相独立)，两个启动环境间的切换是两条时间线的穿越（或者说平行空间的穿越）。默认安装中 `zroot/ROOT/default` 是默认的启动环境。

```shell-session
# zfs snap zroot/ROOT/default@new                         # 建一个 zfs 快照
# zfs clone zroot/ROOT/default@new zroot/ROOT/new         # 用刚建的快照复制一个镜像
```

复制的镜像可以作为一个启动环境，可以用bectl工具查看可用的启动环境

```shell-session
# bectl list
BE                                Active Mountpoint Space Created
0915                              -      -          4.00M 2023-09-19 19:44
13.2-RELEASE-p2_2023-09-13_141111 -      -          29.0M 2023-09-13 14:11
new                               -      -          432K  2023-09-20 15:17
default                           NR     /          40.8G 2023-04-10 10:06
```

其中 Active 列中 `N` 表示当前使用环境，`R` 表示下次启动时使用的环境。bectl 工具可以改变下次使用的启动环境（在启动 FreeBSD 时，启动菜单里选 `8`，也可以改变启动环境）

```shell-session
bectl activate new
Successfully activated boot environment new
```

再次用 `bectl list` 查看，观察 Active 列的变化

```shell-session
bectl list
BE                                Active Mountpoint Space Created
0915                              -      -          4.00M 2023-09-19 19:44
13.2-RELEASE-p2_2023-09-13_141111 -      -          29.0M 2023-09-13 14:11
new                               R      /          2.84M 2023-09-20 15:17
default                           N      -          40.8G 2023-04-10 10:06
```

重启 FreeBSD （启动菜单里选择 new 启动环境，或如上用 `bectl activate new` 切换到 new 启动环境），用 df 观察，挂载的根目录的文件系统已经是 `zroot/ROOT/new`

```shell-session
# df
Filesystem          1K-blocks     Used     Avail Capacity  Mounted on
zroot/ROOT/new      110611616 42612156  67999460    39%    /
devfs                       1        1         0   100%    /dev
/dev/gpt/efiboot0      266176     1872    264304     1%    /boot/efi
fdescfs                     1        1         0   100%    /dev/fd
```

切换回 `zroot/ROOT/default` 启动环境，在启动菜单里选择 default 启动环境，或如上用 `bectl activate default` 切换到 default 启动环境

用法扩展：可以把一个启动环境升级为 FreeBSD 14，实现 13、14 多版本共存。

参考文献：

- [wiki/BootEnvironments](https://wiki.freebsd.org/BootEnvironments)

## FreeBSD on zfs 的 zpool 升级

13.2 升级 14.0 ，zpool 版本有升级。

此处假定已经用 `freebsd-update` 从 13.2 升级到 14.0。

开始前的提醒：准备好 livecd 以应对意外，livecd 要 14.0 及以上的，13.2 不支持（不能访问） 14.0 的 zfs

查看 zpool 状态：

```shell-session
root@u13t14 # zpool status

  pool: zroot
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
the pool may no longer be accessible by software that does not support
the features. See zpool-features(7) for details.
config:

NAME        STATE     READ WRITE CKSUM
zroot       ONLINE       0     0     0
  ada0p3    ONLINE       0     0     0

errors: No known data errors
```

未升级前不能使用所有 zfs 新功能，下面进行升级：

```shell-session
root@u13t14 # zpool upgrade zroot

This system supports ZFS pool feature flags.

Enabled the following features on 'zroot':
  edonr
  zilsaxattr
  head_errlog
  blake3
  block_cloning
  vdev_zaps_v2

Pool 'zroot' has the bootfs property set, you might need to update
the boot code. See gptzfsboot(8) and loader.efi(8) for details.
```

### 重写引导（仅非 EFI 引导需要）

**此处提示重要**
>
>bootfs 属性是在 zfs 上引导 FreeBSD 的重要标志，不理睬这个提示可能没事，但出了问题就不能引导系统，建议按提示重写 `boot code` (为什么这么建议？因为我炸了)。如果你没有 **freebsd-boot** 分区就不需要以下操作。

查看分区信息：

```shell-session
root@u13t14 # gpart show

=>      40  33554352  ada0  GPT  (16G)
        40      1024     1  freebsd-boot  (512K)
      1064       984        - free -  (492K)
      2048   4194304     2  freebsd-swap  (2.0G)
   4196352  29356032     3  freebsd-zfs  (14G)
  33552384      2008        - free -  (1.0M)
```

找到 `freebsd-boot` 类型分区，这里序号为 1，对应下面命令中 `-i` 选项，接着重写 boot code 

```shell-session
root@u13t14 # gpart bootcode -p /boot/gptzfsboot -i 1 ada0
partcode written to ada0p1
```

可再次查看 zpool 状态：

```shell-session
root@u13t14 # zpool status

  pool: zroot
 state: ONLINE
config:

NAME        STATE     READ WRITE CKSUM
zroot       ONLINE       0     0     0
  ada0p3    ONLINE       0     0     0

errors: No known data errors
```


  
## 注意事项

- ZFS 并不使用 `/etc/fstab`，但是 EFI、Swap 仍然使用。
