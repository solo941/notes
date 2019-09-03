## 仓库

![pic](https://github.com/solo941/notes/blob/master/Git/pics/微信图片_20190903180416.jpg)

1. **Remote:** 远程主仓库；
2. **Repository/History：** 本地仓库；
3. **Stage/Index：** Git追踪树,暂存区；
4. **workspace：** 本地工作区（即你编辑器的代码）

![pic](https://github.com/solo941/notes/blob/master/Git/pics/微信图片_20190903180447.jpg)

## 代码提交流程

一般代码提交流程为：**工作区** -> `git status` 查看状态 -> `git add .` 将所有修改加入**暂存区**-> `git commit -m "提交描述"` 将代码提交到 **本地仓库** -> `git push` 将本地仓库代码更新到 **远程仓库**

### 使用场景：

#### git add . 之后希望丢弃修改：

git checkout -- .

#### commit 信息修改

git commit --amend -m“新提交消息”

#### commit 遗漏更新

git add missed-file // missed-file 为遗漏提交文件
git commit --amend --no-edit

#### 删除指定的commit,回退版本

#### git reset

 git reset的作用是修改HEAD的位置，即将HEAD指向的位置改变为之前存在的某个版本

```
清除commit的内容
git reset -- .
//回退一个版本,不清空暂存区,将已提交的内容恢复到暂存区,不影响原来本地的文件(未提交的也不受影响) 
git reset –soft HEAD~1  
//回退一个版本,清空暂存区,将已提交的内容的版本恢复到本地,本地的文件也将被恢复的版本替换
git reset –hard HEAD~1
```

其中的HEAD~1 可以使用任意版本号，通过git log查看

#### git revert

`git revert`是提交一个新的版本，将需要`revert`的版本的内容再反向修改回去，版本会递增，不影响之前提交的内容。比如，我们commit了三个版本（版本一、版本二、 版本三），突然发现版本二不行（如：有bug），想要撤销版本二，但又不想影响撤销版本三的提交，就可以用 git revert 命令来反做版本二，生成新的版本四，这个版本四里会保留版本三的东西，但撤销了版本二的东西。

```
//使用“git revert -n 版本号”命令。如下命令，我们反做版本号为8b89621的版本
git revert -n 8b89621019c9adc6fc4d242cd41daeb13aeb9861
//提交，使用“git commit -m 版本名”
git commit -m "revert add text.txt" 
git push
```

## 分支

- 创建本地分支 `git branch 分支名`
- 查看本地分支 `git branch`
- 查看远程分支 `git branch -a`
- 切换分支  `git checkout 分支名`
- 合并本地分支 `git merge hotfix`：(将 hotfix 分支合并到当前分支)
- 合并远程分支 `git merge origin/serverfix`

### 在本地将 feature 分支合并到 dev 分支

```
git checkout dev
git pull # 更新 dev 分支
git log feature..dev
```

如果没有输出任何提交信息的话，即表示 feature 对于 dev 分支是 up-to-date 的,可以直接合并。

否则，需要rebase，将feature拼接到dev之后再合并。

```
git checkout feature
git rebase dev
```

### 同步Github fork 出来的分支

1、配置remote，指向原始仓库

```
git remote add upstream https://github.com/InterviewMap/InterviewMap.git
```

2、上游仓库获取到分支，及相关的提交信息，它们将被保存在本地的 upstream/master 分支

```
git fetch upstream
# remote: Counting objects: 75, done.
# remote: Compressing objects: 100% (53/53), done.
# remote: Total 62 (delta 27), reused 44 (delta 9)
# Unpacking objects: 100% (62/62), done.
# From https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY
# * [new branch] master -> upstream/master
```

3、切换到本地的 master 分支

```
git checkout master
# Switched to branch 'master'
```

4、把 upstream/master 分支合并到本地的 master 分支，本地的 master 分支便跟上游仓库保持同步了，并且没有丢失本地的修改。

```
git merge upstream/master
# Updating a422352..5fdff0f
# Fast-forward
# README | 9 -------
# README.md | 7 ++++++
# 2 files changed, 7 insertions(+), 9 deletions(-)
# delete mode 100644 README
# create mode 100644 README.md
```

5、上传到自己的远程仓库中

```
git push 
```

### 解决 git pull 时的冲突

#### **未commit先pull，视本地修改量选择revert或stash**

如果本地修改量小，例如只修改了一行，可以按照以下流程

```
-> revert(把自己的代码取消) -> 重新pull -> 在最新代码上修改 -> [pull确认最新] -> commit&push
```

 本地修改量大，冲突较多

```
-> stash save(把自己的代码隐藏存起来) -> 重新pull -> stash pop(把存起来的隐藏的代码取回来 ) -> 代码文件会显示冲突 -> 右键选择resolve conflict -> 打开文件解决冲突 ->commit&push
```

#### **已commit未push，视本地修改量选择reset或直接merge**

如果本地修改量小，例如只修改了一行，可以按照以下流程  

```
-> reset(回退到未修改之前，选hard模式，把自己的更改取消) -> 重新pull -> 在最新代码上修改 -> [pull确认最新] -> commit&push
```

#### **修改量大，直接merge，再提交（目前常用）**

```
-> commit后pull显示冲突 -> 手动merge解决冲突 -> 重新commit -> push
```



## 参考资料

[珍藏多年的 Git 问题和操作清单](https://mp.weixin.qq.com/s/_jkzxQzQCppADch3CcZM_A)

[**git rebase 还是 merge的使用场景最通俗的解释**](https://www.jianshu.com/p/4079284dd970)

[Git](https://github.com/CyC2018/CS-Notes/blob/master/notes/Git.md)

[git pull时冲突的几种解决方式](https://www.cnblogs.com/zjfjava/p/10280247.html)

