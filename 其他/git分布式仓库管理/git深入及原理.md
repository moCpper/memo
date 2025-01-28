## HEAD
在.git中存在一个HEAD目录，用于保存当前所操作的分支，
![[Pasted image 20250128151942.png]]
查看可得HEAD指向了一个目录，该目录即表示我们所工作的当前分支。
再次查看该目录中master文件的内容：
```bash
cat ./refs/heads/master
out : b00240d35c9470d1293ce211fe6a112491a74bea
```
可以看到这是一段hash值，我们使用git内置的命令查看该hash值所保存的内容后可得
```bash
PS E:\github\.git> git cat-file -t b0024
out : commit                        

PS E:\github\.git> git cat-file -p b0024
tree aeeaf8c727843c32a5e5500b987f13d57b51f6a2
author moCpper <2540698708@qq.com> 1736775019 +0800
committer moCpper <2540698708@qq.com> 1736775019 +0800

PS E:\github\.git> git cat-file -p aeeaf8
040000 tree f894983e3bc5a8a049c47c5e2f3949eba920735b    thread_pool
push
```
由此可得mater文件中所保存的hash值为master分支**最后一次**commit。
```bash
PS E:\github\.git> git log
commit b00240d35c9470d1293ce211fe6a112491a74bea (HEAD -> master, origin/master)
Author: moCpper <2540698708@qq.com>
Date:   Mon Jan 13 21:30:19 2025 +0800

    push
```
## git branch
git branch的基本操作流程如下
```bash
PS E:\github\.git> git branch
* master
PS E:\github\.git> git branch dev
PS E:\github\.git> git branch
  dev
* master
PS E:\github\.git> git log
commit b00240d35c9470d1293ce211fe6a112491a74bea (HEAD -> master, origin/master, dev)
Author: moCpper <2540698708@qq.com>
Date:   Mon Jan 13 21:30:19 2025 +0800

    push
```
使用git branch dev创建dev分支后，该分支自动指向与master分支相同的commit.
`需要注意的是：在git branch dev后，objects目录下并没有生成新的文件,可以理解为分支生成操作只是生成一个指针`
```bash
PS E:\github\.git> tree
E:.
├─hooks
├─info
├─logs
│  └─refs
│      ├─heads
│      └─remotes
│          └─origin
├─objects
│  ├─ae
│  ├─b0
│  ├─f8
│  ├─info
│  └─pack
└─refs
    ├─heads
    ├─remotes
    │  └─origin
    └─tags
```
