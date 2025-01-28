
## 关于blob对象
Blob（Binary Large Object）是 Git 存储文件内容的基本单位。每个 Blob 对应一个文件的内容（例如 `README.md` 的文本内容或一张图片的二进制数据），但 **不包含文件名、路径或元数据**（这些信息由 Tree 对象管理）。
### 哈希值的生成原理
- **输入内容唯一性**：Git 将文件内容（字节流）和头部信息（如 `blob <文件大小>\0`）拼接后，通过 SHA-1 算法生成唯一的 40 位十六进制哈希值。
    例如：
    文件内容："Hello World"
    实际哈希输入： "blob 11\0Hello World"
    生成的 SHA-1： 557db03de997c86a4a028e1ebd3a1ceb225be238
- **确定性**：相同内容必然生成相同的哈希值，不同内容几乎不可能碰撞（哈希冲突概率极低）。
### 为什么用哈希值作为文件名？
#### （1）**内容寻址**

- **内容唯一标识**：哈希值直接反映文件内容，确保：
    
    - 文件内容不变 → 哈希不变。
        
    - 文件内容改变 → 哈希必变。
        
- **快速查找与去重**：Git 只需通过哈希即可判断文件是否已存在存储系统中，避免重复存储相同内容。
#### （2）**数据完整性校验**

- **防篡改**：任何对文件内容的修改都会导致哈希值变化，因此可以快速检测到意外损坏或恶意篡改。
    
- **自我验证**：Git 对象（包括 Blob、Tree、Commit）的哈希值天然具备校验功能。
#### （3）**存储结构优化**

- **扁平化存储**：所有 Blob 对象存储在 `.git/objects` 目录中，**哈希值前两位作为目录名，后 38 位作为文件名**。例如：

    哈希值：557db03de997c86a4a028e1ebd3a1ceb225be238
    存储路径：.git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238
    
    这种结构既能避免单目录文件过多，又能高效定位对象。
Blob 使用哈希值作为文件名，是 Git 实现 **内容寻址存储** 和 **高效版本控制** 的核心机制。这种设计确保了数据的完整性、去重能力和快速访问，同时将内容与元数据解耦，为分布式协作和代码管理提供了底层支持。

**需要注意的是：内容在写入 `.git/objects` 存储目录前git会对其进行压缩。**
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
out : commit                        // 表示这是一次commit

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
然后使用git branch --delete dev删除dev分支
```bash
$ git branch --delete dev
Deleted branch dev (was b00240d).
```
**当删除分支后，之前在此分支上产生的tree，commit或是blob依旧存在。**
```bash
刘家豪@LAPTOP-71AV808L MINGW64 /e/github (master)
$ git cat-file -p b00240d
tree aeeaf8c727843c32a5e5500b987f13d57b51f6a2
author moCpper <2540698708@qq.com> 1736775019 +0800
committer moCpper <2540698708@qq.com> 1736775019 +0800

push

刘家豪@LAPTOP-71AV808L MINGW64 /e/github (master)
$ git cat-file -p aeeaf8c
040000 tree f894983e3bc5a8a049c47c5e2f3949eba920735b    thread_pool
```

## detached HEAD
**Detached HEAD（分离头指针）** 是 Git 中的一个特殊状态，表示 `HEAD` 指针直接指向某个具体的 **提交（Commit）**，而非指向一个 **分支（Branch）**。此状态常见于查看历史提交或临时实验代码时。
在checkout后跟上具体的commit哈希即可实现：
```bash
git checkout b00240d35c947
```
建议当HEAD指向某个特定commit时，创建新的分支进行操作，如：
```bash
git checkout -b tmp
```
此时新的分支上HEAD指向了这次commit。
当分支被误删除后，分支所产生的commit并没有被删除，此时使用这种方式可以快速定位到之前没来得及操作的commit上。
## 关于远程仓库
在使用git remote add origin master添加远程仓库的master分支后，会在本地同步生成目录**refs/remotes**
以及**logs/refs/remotes**
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
可以看到，在refs目录中生成了一个与自定义的远程仓库名称相同的目录refs/remotes/origin，
并在该目录下生成了一个与分支同名的文件master。通过命令查看该文件可得：
```bash
PS E:\github\.git> cat .\refs\remotes\origin\master
b00240d35c9470d1293ce211fe6a112491a74bea
PS E:\github\.git> git cat-file -p b00240d
tree aeeaf8c727843c32a5e5500b987f13d57b51f6a2
author moCpper <2540698708@qq.com> 1736775019 +0800
committer moCpper <2540698708@qq.com> 1736775019 +0800

push
```
可以得出，**master中保存的是一个commit对象**，且远程仓库的commit记录与本地同步，可以通过git log验证这一点
```bash
PS E:\github\.git> git log
commit b00240d35c9470d1293ce211fe6a112491a74bea (HEAD -> master, origin/master)
Author: moCpper <2540698708@qq.com>
Date:   Mon Jan 13 21:30:19 2025 +0800

    push
```
若在本地commit之后，远程master分支与本地master不同步，用git log验证，效果可能如下：
![[Pasted image 20250128212959.png]]
通过git push origin master实现同步
