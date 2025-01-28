# git的基本使用
****
**git clone ssh** clone远程仓库地址,
本地自动生成:
- **远程仓库名称origin**
- 默认的主**干分支master**
`本地主干分支**master**追踪到**远程origin仓库**的**master分支**`
## **git基本流程：**
`git工作区  git add=》 暂存区 git commit =》 本地仓库的代码分支，本地的master分支`
如：
![[Pasted image 20240908211241.png]]
或：
![[Pasted image 20240908213044.png]]
注：**远程仓库默认名称为origin

 **git push origin master : master**
将本地**master**分支上的代码提交到远程**origin仓库**的**master分支**上，origin后跟本地仓库分支名。
可缩写为：git push origin master，表示将本地maste推送到远程origin仓库.
**注：在push之前最好pull一下保持当前分支与其远程分支的同步**
# git各阶段版本回退命令
****
 **git add之前（工作区修改的代码）**
工作区修改的代码不要了：
```test
**git checkout** -- < name >
```
《本地仓库/分支》=》 《当前工作区（本地文件夹）修改的内容/文件》覆盖掉  

** git add之后（处于暂存区）**
** git reset HEAD < name >.. ** 用以取消暂存

** git commit 之后（处于本地仓库）**
** git reset -- hard 《commit - id》**
- 将HEAD指针**移到**指定的commit  id
- git reflog 查看HEAD的改动日志
注： **`HEAD 指向最后（最新）一次commit提交`**

# 解决git推送冲突问题
`git pull` 命令会尝试自动整合远程分支的更改到你的当前分支。具体来说，`git pull` 实际上是两个命令的组合：`git fetch` 和 `git merge`。
1. **`git fetch`**：从远程获取更新，但不进行合并。它将更新本地的远程分支信息。
2. **`git merge`**：把获取的更改合并到你的当前分支。

**git pull**拉取远程仓库代码后
你的本地工作区代码会如下：
![[Pasted image 20240908224622.png]]
`<<<<<<< HEAD` 的标记意味着你遇到了冲突。这是 Git 用来指示合并过程中发生的冲突的标记。
- `<<<<<<< HEAD`：表示当前分支的代码。
- `=======`：这部分是分隔符，用来区分两段代码。
- `>>>>>>> [其他分支]`：表示你要合并的分支中的代码。
# 分支版本控制
- git branch
	- 查看本地当前分支
- git branch -r 
	  -  查看远程分支
- git branch -a
	- 查看本地和远程分支
- git branch -vv
	-  一个非常有用的命令，用于列出**本地分支及其相关的远程跟踪分支**，输入内容为：
		1. **当前分支**：前面有 `*` 的分支表示你当前所在的分支。
		2. **本地分支名称**：显示所有本地分支的名称。
		3. **远程跟踪分支**：与本地分支关联的远程分支（如果有）

## git checkout
****
用于创建分支
**git checkout -b  < name >**
可分开为
- git branch <  >
- git checklout <  >
用于创建新分支并切换

**git merge sortdev**
将sortdev分支上的代码合并到**当前分支的仓库和工作区**中

### 分支冲突
这里以 `git merge`举例。
原始代码只有一个 README.md 文件，内容为：

```text
This is first commit.
```

我在 master 分支上将该文件进行如下修改并提交：

```text
This is second commit.
```

然后通过命令 `git checkout -b dev` ，新建一个 dev 分支，并切换到该分支。

修改 README.md 文件为如下内容，并进行提交：

```text
This is third commit.
```

我这里在两个分支上，对同一个文件同一行进行了修改并提交。

然后切换到 master 分支上，通过命令 `git merge dev` ，将两个分支进行合并。

这时会提示 README.md 文件发生了冲突：

```text
$ git merge dev
Auto-merging README.md
CONFLICT (content): Merge conflict in README.md
Automatic merge failed; fix conflicts and then commit the result.
```

README.md 文件中的内容自动变成：

```text
<<<<<<< HEAD
This is second commit.
=======
This is third commit.
>>>>>>> dev
```

通过 ======= 将发生冲突的两部分进行隔开，表示这两部分不同内容会在文件中的相同位置。

HEAD 表示当前本地所在分支的别名，dev 表示将要将要合并过来的分支。

手动修改 READ.md 文件去掉冲突，比如修改为：

```text
This is second/third commit.
```
 
 将本地分支追踪远程仓库分支
 ```test
**git checkout -b < name > origin/< name >**
```

总结：
**查看远程仓库名称  一般远程仓库名称默认为origin**
```test
git remote
```

** 查看本地分支**
```test
git branch
```

**查看远程分支**
```test
git branch -r
```

**查看本地分支和远程分支的追踪关系**
```test
git checkout -vv
```

**创建本地分支并指定追踪哪个远程分支**
```test
git checkout -b <本地分支名><远程仓库名>/<远程分支名>
```

设置已经存在的本地分支追踪哪个远程分支
```test
git branch -u <远程仓库名>/<远程分支名>
```

**git基本工作流**
![[Pasted image 20240909201658.png]]
![[Pasted image 20240909203636.png]]
![[Pasted image 20240909203900.png]]

**release阶段bug修改**
```test
1. git checkout -b bugfix/mybug origin/release
2. 修改bug
3. git add xxxx
4. git commit -m "xxxx"
5. git pull
6. git push origin bugfix/mybug
7. 代码评审工具 bugfix/mybug =》 release MR 拉人 CR merge =》 release 分支上
8. git push origin : bigfix/mybug
```
