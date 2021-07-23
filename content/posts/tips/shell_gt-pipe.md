---
title: shell中的 >| 是干啥的???
author: ash
tags: ["tips", "shell"]
categories: ["Tips"]
date: 2021-07-12T18:51:10+08:00
cover: "/images/vr.jpg"
---

```sh
$ bash pkgbuild.sh -t s100gt -s /mnt/data/product_demo/build/rootfs/ -v x.x.x_xx -o /mnt/data/product_demo/build/output >| /mnt/data/product_demo/build/pkg.log 2>&1
```

It's not useless - it's a specialised form of the plain > redirect operator (and, perhaps confusingly, nothing to do with pipes). bash and most other modern shells have an option noclobber, which prevents redirection from overwriting or destroying a file that already exists. For example, if noclobber is true, and the file /tmp/output.txt already exists, then this should fail:

```sh
$ some-command > /tmp/output.txt
```

However, you can explicitly override the setting of noclobber with the >| redirection operator - the redirection will work, even if noclobber is set.

You can find out if noclobber is set in your current environment with set -o.

For the historical note, both the "noclobber" option and its bypass features come from csh (late 70s). ksh copied it (early 80s) but used >| instead of >!. POSIX specified the ksh syntax (so all POSIX shells including bash, newer ash derivatives used as sh on some systems support it). Zsh supports both syntaxes. I don't think it was added to any Bourne shell variant but I might be wrong.
