---
title: Rsync under the hood
author: ash
tags: ["linux", "rsync", "unix"]
categories: ["UNIX哲学"]
date: 2021-04-22T15:35:01+08:00
image: chino00.png
---

## **0x00 rsync 简介**

`Rsync` 是一款开源，快速，多功能的数据同步工具.

* `rsync` 可以实现本地与远程主机之间数据的快速同步，本地主机的不同分区或目录之间也能进行同步
* `rsync` 实现了全量或增量的数据同步、远程备份，类似 `scp`
    > _但是 `scp` 只能全量备份_
* `rsync` 同样可以用于文件和目录的删除
* `rsync` 以一当三 实现了 `scp` `cp` `rm` 的功能，且优于它们

## **0x01 rsync 核心算法**

### **前情提要**

在开始算法原理之前，先简单说明下 `rsync` 是如何进行增量传输的.

假设现有主机和文件信息如下:

||主机名|文件名|文件内容|
|--:|:-:|:-:|---|
|发送端|α|A|ashxx123_zoe|
|接收端|β|B|ash123zoey|

我们要将文件 A 同步到 β 主机上
> _实际上 A 和 B 是同名文件, 为区分使用不同的名字_

* 如果 β 上不存在A文件，那么 `rsync` 会直接传输 A 文件
* 如果 β 上的目标路径下已经存在 A 文件，那么 α 端会根据具体情况觉得是否传输 A 文件
    1. `rsync` 使用 "quick check" 算法，比较 A 与 B 的size，mtime 等元数据信息，如果不相同则进行同步，否则忽略
    2. 如果确定要传输 A 文件，会进一步进行比较，只传输 A 与 B 不同的部分，实现真正的增量传输

现在假设 α 上有文件A，β 上有文件B，如何让 A、B 同步呢? 我们的第一反应直截了当，把A直接复制到β上，不就完事了嘛~
> 但是仔细一想，如果文件A很大，并且A和B高度相似，复制整个文件消耗大量的时间和系统资源

现在对比一下 A 和 B，发现有大量相似的数据 `ash`，`123`，`zoe`，A文件多出了 `xx` 和 `_`, B文件多个 `y`

`rsync`，会将 `xx` 和 `_` 发送给 β主机，对于相同部分，则会直接从 B文件中复制，最终合成一个A的副本，重命名并覆盖掉B文件, 完成数据的同步

有了这些基本的概念，我们就可以更进一步，刨一刨 `rsync` 的增量传输算法的实现.

### **正片开启**

依然拿之前的数据来说，`rsync` 将发送端α 上的 文件A 同步到 接收端β， 主要步骤:

1. α 通知 β 文件A 等待传输
2. β 收到信息后，将文件B 划分成一系列大小固定的数据块(chunk)，对chunk编号，同时记录chunk的起始偏移地址及数据长度

    > _最后一个chunk可能较小_
    > 此处假设chunk大小为 3字节，那么文件B `ash123zoey` 将被划分为以下几个chunk

    | count=4 n=3 rem=1 | 表示划分了4个数据块，数据块大小为3字节，剩余1字节给了最后一个数据块 |
    |---|---|
    | chunk[0]：offset=0 len=3 | 该数据块对应的内容为ash |
    | chunk[1]：offset=3 len=3 | 该数据块对应的内容为123 |
    | chunk[2]：offset=6 len=3 | 该数据块对应的内容为zoe |
    | chunk[3]：offset=9 len=1 | 该数据块对应的内容为y   |

3. β 对文件B 的每一个chunk 都根据内容计算两个校验码，将计算出的校验码跟随在对应chunk后组成校验码集合，发送给 α 端

    > 两个校验码分别是:
    * 32位的弱滚动校验码(rolling checksum)
    * 128位的MD5强校验码

    > 校验码集合大致如下：其中sum1为rolling checksum，sum2为md5sum

    | 数据块 | rolling checksum | md5 checksum |
    |---|---|---|
    | chunk[0] | sum1=3ef2c827 | sum2=3efa923f8f2e7 |
    | chunk[1] | sum1=57ac2aaf | sum2=aef2dedba2314 |
    | chunk[2] | sum1=92d7edb4 | sum2=a6sd6a9d67a12 |
    | chunk[3] | sum1=afe74939 | sum2=90a12dfe7485c |

    > 需要注意，不同内容的 chunk 计算出的rolling checksum是有可能相同的

4. α 接收到文件B 的校验码集合后，对集合中的每个 rolling checksum 计算16位长度的hash值，并将每 $2^{16}$ 个hash值按照hash顺序放入一个hash表中，hash表中的每一个hash条目都指向校验码集合中它所对应的 rolling checksum 的chunk号，然后对校验码集合根据 hash 值进行排序，这样排序后的校验码集合中的顺序就能和 hash表中的顺序对应起来

    > 所以，hash表和排序后的校验码集合对应关系大致如下：假设hash表中的hash值是根据首个字符按照[0-9a-z]的顺序进行排序的。

    |弱滚动校验码| 散列表 | -> | 数据块 | 弱滚动校验码 | 强校验码 |
    |---|---|---|---|---|---|
    | sum1 = 57ac2aaf | 2aaf | ---> | chunk[1] | sum1=57ac2aaf | sum2=aef2dedba2314 |
    | sum1 = afe74939 | 4939 | ---> | chunk[3] | sum1=afe74939 | sum2=90a12dfe7485c |
    | sum1 = 3ef2c827 | c827 | ---> | chunk[0] | sum1=3ef2c827 | sum2=3efa923f8f2e7 |
    | sum1 = 92d7edb4 | edb4 | ---> | chunk[2] | sum1=92d7edb4 | sum2=a6sd6a9d67a12 |

5. α 对文件A 进行处理，从第 1 个字节开始取大小相同的 chunk 并计算它的校验码，去校验码集合中匹配：
    * 如果匹配到某个chunk条目，表示该 chunk 和文件B 中数据块相同，因此不必传输，于是主机α 跳转到该 chunk 的尾偏移地址，从此偏移处继续取 chunk 进行匹配
    * 如果不能匹配到校验码集合中的 chunk 条目，则该 chunk 需要传输给β，于是 α 跳转到下一个字节处继续取 chunk 进行匹配
    > PS:匹配成功时跳过的是整个匹配数据块，匹配不成功时跳过的仅是一个字节

6. 当 α 发现 chunk 匹配时，只发送这个匹配块的附加信息给 β. 如果两个匹配块之间有非匹配的数据，会发送这些数据. 当β 收到数据后，会创建一个临时文件，并通过这些数据重组这个临时文件，使其内容和A文件相同.
    > 临时文件重组完成后，修改临时文件的属性信息(如权限、所有者、mtime等)，然后重命名该临时文件替换掉文件B，完成文件同步.

### **示例分析**

没错，刚才数据的价值还没有被榨干！！

|弱滚动校验码| 散列表 | -> | 数据块 | 弱滚动校验码 | 强校验码 | 内容 |
|---|---|---|---|---|---|--:|
| sum1 = 57ac2aaf | 2aaf | ---> | chunk[1] | sum1=57ac2aaf | sum2=aef2dedba2314 | 123 |
| sum1 = afe74939 | 4939 | ---> | chunk[3] | sum1=afe74939 | sum2=90a12dfe7485c | y |
| sum1 = 3ef2c827 | c827 | ---> | chunk[0] | sum1=3ef2c827 | sum2=3efa923f8f2e7 | ash |
| sum1 = 92d7edb4 | edb4 | ---> | chunk[2] | sum1=92d7edb4 | sum2=a6sd6a9d67a12 | zoe |

对于文件A，`ashxx123 zoe`
对于文件B，`ash123zoey`
对于文件A，`ashxx123 zoe`，从第一个字节开始, 取3字节长度的 chunk -> `ash`, 去校验码集合中比对，
> 由于和 文件B 的 chunk[0] 内容完全相同，所以 rolling-checksum 与 对应的 hash值 也一定相同.

计算 `ash` 的rolling-checksum 及对应的 hash值，在hash表中匹配成功，进入第二层级的匹配
进行rolling-checksum 的比较，从 hash值指向的 chunk[0] 的条目处开始向下扫描，扫描过程中发现扫描的第一条信息就能匹配上，所以扫描终止，进入第三层的搜索匹配
α 计算 `123` 的强校验码， 与 chunk[0] 对应的强校验码比较，最终发现能匹配上，于是确定了文件A 中的 `123` 数据块是匹配数据块，不需要传输给 β.

> 虽然匹配数据块不用传输，但匹配的相关信息需要立即传输给 β，否则 β 不知道如何重组文件
> 匹配块需要传输的信息包括：匹配的是文件B 中的 chunk[0]，在文件A 中该数据块的起始偏移地址为第1个字节，长度为3字节

数据块 `123` 的匹配信息传输完成后，α 主机将取第二个数据块进行处理，即从第4个字节开始取数据块，所以 α 取得的第2个数据块内容为 `xx1`，同样，需要计算它的rolling checksum和 对应的hash值，并搜索匹配 hash表中的 hash条目，发现没有任何一条 hash值可以匹配上，于是立即确定该数据块是非匹配数据块

α 继续向前取文件A 中的第三个数据块进行处理，由于第二个数据块没有匹配上，所以取第三个数据块时只跳过了一个字节的长度，即从第5个字节开始，取得数据块内容为 `xab`，比较发现，这个数据块和第二个数据块一样是无法匹配的数据块，于是继续向前跳过一个字节，即从第6个字节开始，取得数据块内容为 `abc`，这个数据块是匹配块，所以和第一个数据块一样处理
> 第一个数据块到第四个数据块，中间有两个非匹配数据，于是在确定第四个数据块是匹配块后，会将中间的非匹配内容(即123xxabc中间的xx)逐字节发送给 β

依此处理完A中所有数据，最终有3个匹配数据块 chunk[0]、chunk[1] 和 chunk[2]，以及 2段 非匹配数据 `xx` 和 `_`

这样 β 就收到了匹配数据块的匹配信息以及逐字节的非匹配纯数据，这些数据是 β 重组文件A 副本的关键信息，它的大致内容如下：
||
|---|
|chunk[0] of size 3 at 0 offset=0|
|data receive 2 at 3|
|chunk[1] of size 3 at 3 offset=5|
|data receive 1 at 8|
|chunk[2] of size 3 at 6 offset=9|
||

||||||||||||||
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| A | a | s | h | x | x | 1 | 2 | 3 | _ | z | o | e |
| offset | 0 | 1| 2| 3| 4| 5| 6| 7| 8| 9| 10| 11 |
| B | a | s | h | 1 | 2 | 3 | z | o | e | y |  |  |
| offset | 0 | 1| 2| 3| 4| 5| 6| 7| 8| 9| | |
||||||||||||||

* `chunk[0] of size 3 at 0 offset=0`，表示这是一个匹配数据块，匹配的是文件B 中的 chunk[0]，块大小为3字节，`at` 表示这个匹配块在文件B 中的起始偏移地址为0，`offset` 表示这个匹配块在文件A 中起始偏移地址也为0，它也可以认为是重组临时文件中的偏移. 也就是说，在β 重组文件时，将从文件B 的`at 0`偏移处拷贝长度为3字节的chunk[0]对应的数据块，并将块内容写入到临时文件中的offset=0偏移处，这样临时文件中就有了第一段数据`ash`.

* `data receive 2 at 3`，这一段表示这是接收的纯数据信息，不是匹配块，2表示接收的数据字节数，`at 3` 表示在临时文件的起始偏移3处写入这两个字节的数据，这样临时文件就有了包含了数据`ashxx`

* `chunk[1] of size 3 at 3 offset=5`，表示匹配数据块从文件B的起始偏移地址 `at 3` 处拷贝长度为3字节的 chunk[1] 对应的数据块，并将块内容写入临时文件的起始偏移 `offset 5` 处，这样临时文件就有了 `ashxx123`

* `data receive 1 at 8`，接收纯数据信息，表示将接收到的1个字节的数据写入到临时文件的起始偏移地址8处，所以临时文件中就有了`ashxx123_`

* `chunk[2] of size 3 at 6 offset=9`，表示从文件B 的起始偏移地址 `at 6` 处拷贝长度为3字节的 chunk[2] 对应的数据块，并将块内容写入临时文件的起始偏移 `offset 9` 处，这样临时文件就包含了`ashxx123 zoe`

到此为止，临时文件就重组结束了，它的内容和 α 上文件A 内容是完全一致的，然后只需将此临时文件的属性修改一番，并重命名替换掉文件B 即可，这样就完成了文件B 和文件A 的同步

## chunk大小选择

chunk的大小会影响rsync算法的性能，所以划分合适的chunk大小非常重要，默认情况下，rsync会根据文件大小自动判断数据块大小

* 如果尺寸太小，chunk数量会非常多，需要计算和匹配的校验码也会增加，性能变差，同时出现hash值重复、rolling checksum重复的可能性也增大

* 如果尺寸太大，则可能出现大量chunk无法匹配的情况，导致这些数据块都被传输，降低了增量传输的优势

* `rsync` 通过 `-B` [`--block-size`] 选项支持自定义chunk大小，建议大小设定在500-1000字节之间

## **0x02 rsync 缺陷**

看起来 `rsync` 果然神通广大，那 ta 是否真的完美无缺呢？

1. 性能问题！？  服务端cpu，内存消耗巨大 （4倍 计算量）

2. 安全问题！？  dos隐患

3. 压缩问题！？  压缩文件 增量传输 失效？

## **0x03 rsync 场景分析**

根据上文我们知道，`rsync` 增量传输时，sender端因为要多次计算、多次比较各种校验码，因此对cpu的消耗很高，而receiver端需要从基础文件中复制数据，从而需要较高的io消耗.

`rsync` 全量传输时(如第一次同步，或显式使用了全量传输选项 `--whole-file` )，sender端不用计算、比较校验码，receiver端不用复制基础文件，所以资源消耗同 `scp` .

据此，我们得出：

1. `rsync` 对数据库文件的实时同步 考量

    > * 数据库文件通常很大，且更新频繁，使用 `rsync` 实时同步，sender端需要进行大量计算，严重消耗cpu资源，而receiver端由于从巨大的文件中复制大量数据，会让机器承受巨大的io压力
    > * 这种情况下，只适合用 `rsync` 偶尔同步一次进行备份.
    > * 而数据库文件的实时同步，就交给数据库自身的replication功能吧

2. `rsync` 对大量小文件的实时同步 考量

    > * 由于 `rsync` 是增量同步，所以对于receiver端已经存在的与sender端相同的文件，sender端不会发送，这样就使得sender端和receiver端都只需要处理少量的文件，同时由于文件小，所以无论是sender端的cpu还是receiver端的io都不是问题
    > * PS: -> `rsync` 的实时同步需要借助工具来实现 (如inotify+rsync，sersync) 同时这些工具也要设置合理，否则实时同步一样效率低下.

3. `rsync` 并行执行 探究

    > * 在实际业务中，我们有时候需要将一个文件同步到多个地方，这时 `rsync` sender端的cpu压力将无比巨大.
    > * 这种时候就需要...[嘿嘿嘿, 预知后事如何，且听下回分解]

## **rolling checksum alogrithm**

由于需要快速匹配，所以这里的 弱滚动校验 需要有以下性质:
> 当给定 $X_1..X_{n+1}$ 的值以及 $X_1..X_n$ 的弱校验码
> 能迅速得出 $X_2..X_{n+1}$ 的弱校验码

$$a(k, l) = (\sum_{i=k}^l X_i) \ mod \ M$$

$$b(k, l) = (\sum_{i=k}^l (l-i+1) X_i) \ mod \ M$$

$$s(k, l) = a(k,l) + 2^{16}\ b(k,l)$$

此处的 $s(k,l)$ 代表 $X_k..X_l$ 字节的弱滚动校验码, 为了简单与快速，取 $M = 2^{16}$ 这个校验和的重要性质是，可以使用递归关系非常有效地计算连续的值

$$a(k+1,l+1) = (a(k,l)-X_k + X_{l+1}) \ mod \ M $$

$$b(k+1,l+1) = (b(k,l)-(l-k+1)X_k+a(k+1,l+1)) \ mod \ M$$

因此，可以以“滚动”方式计算文件中所有可能偏移的长度为S的块的校验和，每一次的计算量都很少。

## Rsync-Daemon

1. 编辑配置文件

    ```sh
    # rsync --daemon config
    $ vi /etc/rsyncd.conf
    ----------------------
    # Minimal configuration file for rsync daemon.
    # See rsync(1) and rsyncd.conf(5) man pages for help.
    # Do not set "pid file" here.

    #use chroot = yes
    #read only = yes
    uid = root
    gid = root
    use chroot = no
    max connections = 20
    timeout = 300
    mote file = /var/run/rsyncd.mote
    pid file  = /var/run/rsyncd.pid
    lock file = /var/run/rsyncd.lock
    log file  = /var/run/rsyncd.log

    [test01]
    path = /zoe/data
    read only = false
    ```

2. 更改账户密码 + 创建密码文件

   ```sh
   # change passwd & create secret file
   # $ echo "user:passwd" | chpasswd
   $ echo "root:123456" | chpasswd
   ------------------------------------
   $ echo "root:123456" >> /tmp/secrets.file
   ```

3. 启动 `rsync` 守护进程

    ```sh
    # start rsync daemon
    $ rsync --daemon #--ipv4 --ipv6
    ```

4. 同步文件

    ```sh
    # sync data
    $ rsync -avP --password-file=/tmp/secrets.file /path/to/data/  root@ipaddr::test01
    ```

## 参考文献

* [rsync官方技术报告](https://rsync.samba.org/tech_report/tech_report.html)
* [zsync技术paper](http://zsync.moria.org.uk/zsync/paper/)
* [rsync官方文档](https://rsync.samba.org/how-rsync-works.html)
* [rsync简易快照](http://www.mikerubel.org/computers/rsync_snapshots/)
