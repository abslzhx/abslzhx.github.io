+++
title = '使用部分克隆提高 Git 克隆代码的效率'
date = 2023-12-02T12:08:39+08:00
draft = true
+++

源于在使用 Raycast 的一个插件时遇到了报错，但是通过报错信息无法直接定位，所幸 Raycast 商店的插件都开源在 [Raycast-Extension](!https://github.com/raycast/extensions.git) 仓库中，于是打算将插件代码直接拉到本地 Debug 来排查问题。

# 克隆仓库

开始 `clone` 仓库后，发现这个仓库的对象有点多:

```bash
❯ git clone https://github.com/raycast/extensions.git
Cloning into 'extensions'...
remote: Enumerating objects: 96074, done.
remote: Counting objects: 100% (3980/3980), done.
remote: Compressing objects: 100% (701/701), done.
Receiving objects:   0% (647/96074), 21.77 MiB | 7.24 MiB/s
```

通过 `curl https://api.github.com/repos/raycast/extensions` 命令查看返回的仓库信息中 `size` 在 6G 左右，因为 Raycast 官方将所有插件都放在了这个仓库中，所以导致了仓库比较大。

如果完整拉取整个仓库的话会，其实是浪费了时间和存储空间，并不是我想要的，因为我只想要其中的一个插件的代码而已。那有没有办法减少克隆的对象来提高拉取速度呢？

# 浅克隆 (Shallow Clone)

`git clone` 默认情况下会完整拉取仓库的所有分支及提交，但是我们可以指定参数来限制：

- `--branch`: 指定要克隆的分支
- `--depth`: 参数指定克隆的深度，为 1 表示只克隆仓库的最近一次提交的代码

```bash
# clone main 分支，并且限制为最后一次提交
❯ git clone https://github.com/raycast/extensions.git --branch=main --depth=1
Cloning into 'extensions'...
remote: Enumerating objects: 35971, done.
remote: Counting objects: 100% (35971/35971), done.
remote: Compressing objects: 100% (29854/29854), done.
Receiving objects:   2% (777/35971), 26.54 MiB | 5.53 MiB/s
```

可以看到，在限制分支和克隆深度后，克隆的对对象从 `96074` 降到了 `35971`，降低至原来的 37%。如果你只想仓库某个分支的完整代码，并且不关心提交历史，那这个方式能够为你节省很多时间和存储空间。

但是在我这个场景下，我只需要其中的某个目录的代码，所以我们可以进一步的去缩减 clone 的仓库范围。

# 部分克隆 (Partial Clone) 和 稀疏检查 (Sparse Checkout)

## 部分克隆

`git clone` 命令中的以下两个参数，可以进一步限制克隆的范围和克隆后的行为：

```bash
--filter=<filter-spec>
    Use the partial clone feature and request that the server sends a subset of reachable objects according to a
    given object filter. When using --filter, the supplied <filter-spec> is used for the partial clone filter. For
    example, --filter=blob:none will filter out all blobs (file contents) until needed by Git. Also,
    --filter=blob:limit=<size> will filter out all blobs of size at least <size>. For more details on filter
    specifications, see the --filter option in git-rev-list(1).```
-n, --no-checkout
    No checkout of HEAD is performed after the clone is complete.
```

使用部分克隆后：

```bash
❯ git clone --filter=blob:none --no-checkout https://github.com/raycast/extensions.git
Cloning into 'extensions'...
remote: Enumerating objects: 41348, done.
remote: Counting objects: 100% (672/672), done.
remote: Compressing objects: 100% (241/241), done.
remote: Total 41348 (delta 458), reused 557 (delta 415), pack-reused 40676
Receiving objects: 100% (41348/41348), 8.88 MiB | 4.64 MiB/s, done.
Resolving deltas: 100% (20620/20620), done.
```

- `–-filter=blob:none`: 用于过滤仓库中的blob对象，即文件内容。它告诉git在克隆过程中不要将文件的内容下载下来，只克隆文件的元数据（例如文件名、权限、所属文件夹等）。这样可以大大减少克隆的时间和所占用的磁盘空间

– `--no-checkout`: 告诉git在克隆完成后不进行checkout操作，即不将仓库的内容检出到工作目录。这意味着在克隆完成后，你将看不到任何文件在工作目录中。这个参数通常与–filter=blob:none一起使用，用于只获取仓库的元数据而不需要实际文件内容的情况。

通过这两个参数后，可以看到拉取的大小降低到了 8.88M。当我们进入 extensions 目录后，其中只有 .git 文件

```bash
total 0
drwxr-xr-x   96B Dec  2 22:13 .
drwxr-xr-x   96B Dec  2 22:13 ..
drwxr-xr-x  352B Dec  2 22:13 .git
```

## 稀疏检出

接下来，我们要使用 `git sparse-checkout` 来检出我们想要的目录，比如我现在想要检出仓库中的 `extensions/chatgpt` 目录：

```bash
❯ git sparse-checkout set extensions/chatgpt
❯ git checkout main
remote: Enumerating objects: 53, done.
remote: Counting objects: 100% (19/19), done.
remote: Compressing objects: 100% (17/17), done.
remote: Total 53 (delta 3), reused 2 (delta 2), pack-reused 34
Receiving objects: 100% (53/53), 10.18 MiB | 4.19 MiB/s, done.
Resolving deltas: 100% (4/4), done.
Updating files: 100% (53/53), done.
Already on 'main'
Your branch is up to date with 'origin/main'.
```

查看目录中的情况：

```bash
❯ tree -L 3
.
├── CONTRIBUTING.md
├── LICENSE
├── README.md
├── extensions
│   └── chatgpt
│       ├── CHANGELOG.md
│       ├── LICENSE
│       ├── README.md
│       ├── assets
│       ├── metadata
│       ├── package-lock.json
│       ├── package.json
│       ├── src
│       └── tsconfig.json
└── package-lock.json
```

查看仓库大小：

```bash
❯ du -hd1
 10M	./extensions
 25M	./.git
 36M	.
```

可以看到，通过`部分克隆`和`稀疏检出`，我们成功的拉取了我们想要的插件对应目录代码，并且最终的仓库大小只有 36M。

# 参考
- [partial-clone](!https://git-scm.com/docs/partial-clone)
- [Bring your monorepo down to size with sparse-checkout](!https://github.blog/2020-01-17-bring-your-monorepo-down-to-size-with-sparse-checkout/)