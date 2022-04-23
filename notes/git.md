# Git

主要参考****[Version Control (Git)](https://missing.csail.mit.edu/2020/version-control/)****

## Git数据模型

### 区域

Git版本控制系统中共有四个区域，分别是本地工作区域、缓存区域、提交区域以及远程工作区域

- **本地文件工作区域**：包含了项目当前的工作状态，其中的文件有两种状态—tracked和untracked，tracked 文件是那些存在于上一次提交中的文件，即已经在 git 存在过记录的文件，这些文件可能又被修改过，也可能和上次提交时状态一致；Untracked文件则是工作目录下其他所有的文件，如上次提交后在目录中新建的文件

> 可以通过git status查看本地工作区域中的文件状态
> 

> 内容参考**[git 学习记录—— git 中的仓库、文件状态等概念介绍](https://www.cnblogs.com/yhjoker/p/7740203.html)**
> 
- **缓存区域**：包含下一次Commit中修改的文件，方便连续提交多个修改,其中的文件是暂存状态
- **提交区域**：包含了已提交的多个修改记录，构成了一系列提交历史，是DAG图
- **远程工作区域**：是提交区域的一个修改，用于保存在云端

### 文件状态以及转移

git 管理的文件存在以下三种文件状态：

- **committed：**已提交状态，表示数据文件已经被保存至本地数据仓库中。
- **modified：**修改状态，表示文件已被修改，但是尚未被提交(保存)。
- **staged：**暂存状态，表示是被标记了的被修改文件，在下次提交时会将所有标记过的修改保存。

对应上述的三种文件状态可知，若文件已经存在于 git 目录(仓库)中，则文件为**已提交状态**。若文件已被修改并记录至暂存区域，则文件处于**暂存状态**，用户提交时会提交处于此状态的修改。若文件仅仅只是被修改但未被标记为 staged，则为**修改状态**，且在下次提交时并不会保存其相关信息( 必须先进入暂存状态提交时才会保存 )。

而在本地文件工作区域中，在工作目录( working directory )中，存在两种状态的文件，tracked 和 untracked。tracked 文件是那些存在于上一次提交中的文件，即已经在 git 存在过记录的文件，这些文件可能又被修改过，也可能和上次提交时状态一致；Untracked文件则是工作目录下其他所有的文件，如上次提交后在目录中新建的文件等 。当第一次将仓库克隆至目录时，该目录所有文件均为 tracked 且均为未被修改的文件。

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/34f28421-395e-46a5-a75b-d44c4eff57eb/Untitled.png)

### 数据模型

在Git提交区域，将提交历史以DAG图的形式组织成一系列对项目文件的快照，而在每个提交历史中，文件夹—tree,单个文件—blob

```cpp
o <-- o <-- o <-- o
            ^
             \
              --- o <-- o
```

```cpp
<root> (tree)
|
+- foo (tree)
|  |
|  + bar.txt (blob, contents = "hello world")
|
+- baz.txt (blob, contents = "git is wonderful")
```

git中的数据结构共有三种: blob—单个文件，tree—文件夹，commit—一次提交

```cpp
// a file is a bunch of bytes
type blob = array<byte>

// a directory contains named files and directories
type tree = map<string, tree | blob>

// a commit has parents, metadata, and the top-level tree
type commit = struct {
    parents: array<commit>
    author: string
    message: string
    snapshot: tree
}

// 通用的object包括三类
type object = blob | tree | commit
```

git中有两个变量，分别是objects—包含了对象hash-id对对象的映射，references—包含了易懂的名字对hash-id的映射，同时也有一些列针对这两个变量的操作

```cpp
objects = map<string, object>

def store(object):
    id = sha1(object)
    objects[id] = object

def load(id):
    return objects[id]

references = map<string, string>

def update_reference(name, id):
    references[name] = id

def read_reference(name):
    return references[name]

def load_reference(name_or_id):
    if name_or_id in references:
        return load(references[name_or_id])
    else:
        return load(name_or_id)

// 缓存区域
staging_files = map<string, objects>

// 特殊的指针
HEAD
```

Git中各种命令都可转换为上述操作对上述变量的修改以及读取

## Git命令行

理解Git命令行可以从三个角度理解：

- 对本地工作区域做的修改—参考文件状态的改变
- 对git存储的对象做出的修改—参考git数据模型
- 显示结果

### 基础

- git init: 初始化本地文件夹为git repo

  对本地工作区域做出的改变：

1. 其中新建了一个.git文件夹，用于存储git对象
2. 本地文件夹中的所有文件的状态为untracked

对git存储的对象做出的改变:

1. 初始化了三个变量 objects, references, indexes
- git status: 展示git repo的状态
1. 工作区域的文件状态
2. 分支状态
3. 其他
- git add <filename>: 将文件添加到缓存区域
1. <filename>文件状态变为缓存状态
2. 在git 存储的对象来说：将文件添加到indexes中
- git commit: 创建一个提交
1. 所有indexes中的文件变为unmodified
2. 清空indexes, 创建一个快照，更新HEAD指向的分支
- git log --all --graph --decorate: 展示提交历史为一个DAG
- git diff <filename>： 比较当前文件夹与缓存区域的文件有啥不同
- git diff <revision> <filename>：比较提交版本中的文件与当前文件有啥不同
- git checkout <revision>: 更新HEAD为对应的提交版本

### 分支相关

- git branch： 展示分支情况
- git branch <name>: 新建一个分支
- git checkout -b <name>: 新建一个分支并将HEAD更新为新的分支
- git merge <revision>：将<revision>分支合并到当前分支之中
- git mergetool: 用于帮助处理合并冲突
- git rebase：？？？？

### 远程

- git remote：列出分支情况
- git remote add <name> <url>：连接当前工作区域与远程区域，远程区域的名字是<name>
- git push <remote> <local branch>:<remote branch> 将当前分支<local branch>更新到remote的<remote branch>之中
- git branch --set-upstream-to=<remote>/<remote branch>: 设置默认的远程分支，后续直接用git push即可
- git fetch: 下载远程repo中的文件到 <remote>/<remote branch> 之中
- git pull: 下载远程repo,并且合并<HEAD> 分支与<remote>/<remote branch>
- git clone: 下载远程分支

### 撤销

- git reset HEAD <file>: 从缓存区域之中删除当前文件中
- git checkout -- <file>：消除对本地文件的修改

### 其他

- git config：修改配置，或者直接修改~/.gitconfig
- git clone --depth=1: 只要当前版本的
- git add -p：可视化缓存
- git rebase -i: ??
- git blame: 谁修改过
- `git stash`: 移除当前文件夹中的修改
- `git bisect`：搜索提交历史
- • `.gitignore`: 设置不会提交的文件(对git add .非常有用)

## 练习整理

## 常见问题整理

1. 使用ssh连接Github?

(参考**[Using GitHub with SSH](https://www.geeksforgeeks.org/using-github-with-ssh-secure-shell/))**

- 首先在本地生成对应的ssh key,使用命令:

```bash
# 查看本地ssh,如果已经存在就不用再生成ssh key
$ ls -al ~/.ssh
# 如果不存在ssh key,使用ssh-keygen生成本地ssh key
$ ssh-keygen -t rsa -b 4096 -C "YOUR EMAIL"
```

- 复制~/.ssh中的id_rsa.pub的内容到剪切板中

xclip使用参考****[Copy and paste at the Linux command line with xclip](https://opensource.com/article/19/7/xclip)****

```bash
$ xclip -sel clip < ~/.ssh/id_rsa.pub
```

修正：在WSL中，xclip可能无法安装，所以直接用xclip.exe拷贝即可, 参考**[How can I get around using xclip in the Linux subsystem on Win 10?](https://askubuntu.com/questions/1035903/how-can-i-get-around-using-xclip-in-the-linux-subsystem-on-win-10)**

```bash
$ clip.exe  < ~/.ssh/id_rsa
```

- 将剪切板中的内容加入github账户中

**Setting**—>**SSH and GPG keys** —> **New SSH keys**

1. 本地修改开源项目，并把修改的版本添加到自己私有账户中的操作

参考[https://github.com/cmu-db/bustub](https://github.com/cmu-db/bustub)

- 把项目克隆到本地，并且在自己私有账户中新建一个私有项目，把项目源码提交到自己私有项目中

```bash
$ git clone --bare https://github.com/cmu-db/bustub.git bustub-public

# 先新建一个私有项目
$ git push --mirror git@github.com:student/bustub-private.git

# 直接使用私有项目修改相应操作
git clone git@github.com:student/bustub-private.git
```

- 添加现有开源项目的远程连接，用于获取最新的修改

```bash
$ git remote add public https://github.com/cmu-db/bustub.git

# 后续可以用这个命令来获取最新修改
$ git pull public <YOUR BRACH>
```