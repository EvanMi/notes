# GIT & GRADLE

## git

git status 用来查看当前的git的工作空间的情况。

git add 'file' 是一个多功能的命令，可以用来追踪一个新文件，或者把已经追踪的文件放入到暂存区，还能用于合并时把有冲突的文件标记为已解决。

git commit -am '注释' 的作用是省去对已经追踪的且被修改的文件放入到缓存区的一步，直接进行提交。

git commit -m '注释'  将缓存区中的文件提交到本地仓库中。

git的提交信息中会保存用户的<用户名>和<密码>一般配置一下比较好，配置方式：

git config --global user.name 'username'

git config --global user.email 'email'

```
git的config信息可以保存在三个地方：
1、在/etc/gitconfig 对所有的用户生效（几乎不用）--system
2、在~/.gitconfig中（很常用) 也就是 --global
3、针对特定的项目，.git/config文件中 --local
```



git checkout -- '文件' 当修改了文件以后撤销修改, 也可以使用git restore '文件'

git reset HEAD '文件' 将文件从缓存区中移除，也可以使用git restore --staged '文件' 或使用 git rm --cached 'file' 。

git log 查看提交日志

```
-n 用来指定显示的条数
--pretty=oneline
--pretty=format:"%h-%an,%ar:%s"
好用的命令 git log --oneline --graph --all
```

git rm '文件' 删除一个已经提交的文件。

rm '文件' 也可以进行。 两者的区别是， git rm 需要先 git reset HEAD 然后 git checkout --才能恢复，而rm则直接使用 git checkout --就可以恢复。

```
原因是 git rm 做了两件事情：
1、删除了一个文件
2、将被删除的文件放入了缓存区
所以可以直接commit
而 rm则只是删除了文件
```

git mv ’文件1' '文件2'将文件1重命名为文件2，和git rm一样做了两步操作。

git commit --amend -m 'xxxxx' 修改最近一次提交的消息。

.gitigonre文件相关

```
*.b  #表示所有以.b结尾的文件都要进行排除
!a.b #表示a.b例外
/c.d #表示只排除第一级目录下的c.d
/*/x #*表示一层目录
/**/x #**表示任意层次的目录
build/ #忽略build/目录下的所有文件
doc/*.txt #会忽略doc/notes.txt但是不会忽略doc/server/arch.txt
```

git branch #列出当前项目的所有分支

git branch 'newbranch' #会基于当前分支创建一个新的分支

git branch -m 'branch' 'newbranch' #对一个分支进行重命名

git checkout 'newbranch' #切换到对应的分支上

git branch -d 'newbranch' #删除newbranch 

git checkout -b 'new_branch' #创建new_branch并切换过去

git checkout -b 'new branch' 'from branch' # 从指定的分支创建一个新的分支并切换过去

上一条命令新版本的 git推荐使用：git checkout --track origin/test # 其实是上面命令的一种特殊情况，也就是不能指本地分支的名字了。

git branch -v #显示当前分支的最新的版本号

git merge 'branch' #将分支branch切换到当前分支

### git merge的几种情况：

```
Fast-Forward Merge
merge前:
* 0f029c3 (dev) c4
* c594fa9 c3
* 9229adb (HEAD -> master) c2
* cfbff28 c1
merge后：
* 0f029c3 (HEAD -> master, dev) c4
* c594fa9 c3
* 9229adb c2
* cfbff28 c1
加上--no-ff参数后：
*   b735987 (HEAD -> master) c5
|\  
| * 0f029c3 (dev) c4
| * c594fa9 c3
|/  
* 9229adb c2
* cfbff28 c1


Three-Way Merge
merge前：
* 098be39 (HEAD -> master) c5
| * 0f029c3 (dev) c4
| * c594fa9 c3
|/  
* 9229adb c2
* cfbff28 c1
merge后：
*   9c3ee95 (HEAD -> master) c6
|\  
| * 0f029c3 (dev) c4
| * c594fa9 c3
* | 098be39 c5
|/  
* 9229adb c2
* cfbff28 c1

Squash Merge  (--squash)

merge前：
* 098be39 (HEAD -> master) c5
| * 0f029c3 (dev) c4
| * c594fa9 c3
|/  
* 9229adb c2
* cfbff28 c1

merge后：
* 0b7fd35 (HEAD -> master) c6
* 098be39 c5
| * 0f029c3 (dev) c4
| * c594fa9 c3
|/  
* 9229adb c2
* cfbff28 c1


```

git 版本回退

```
git reset --hard HEAD^ 回退到上一个提交，几个^就回退几下
git reset --hard HEAD~n 回退到之前的n个提交
git reset --hard commit_id

git reflog 记录的是操作日志，通过reflog在找回被回退掉的commit，通过reset来跳转
```

git checkout '某个提交（版本号）'会进入一种游离状态

```
在游离状态进行的任何修改都不会对已有的分支产生任何影响，在游离态进行了修改并提交以后，就可以切换到别的分支。这是会有提示，可以使用 git branch 'newbranch' ‘版本号’ 来将刚才的修改生成新的分支。
也可以在游离态的时候，世界使用 git checkout -b ’newbranch'来生成新的分支，然后在该分支上进行修改。

```

git stash相关

```
在当前分支上运行 git stash 会把工作临时保存在一个地方。git stash save 'xxx'可以自定义信息

调用git stash list 来查看所有的保存的工作。可以调用 git stash pop来弹出一个记录；git stash apply 应用状态，但不会删除。 

手动删除可以使用 git stash drop stash@{n}来删除记录。
```

### git标签

```
标签有两种：轻量级标签与带有附注标签
 创建一个轻量级标签 git tag v1.0.1
 创建一个带有附注的标签 git tag -a v1.0.2 -m 'release version'
 删除标签 git tag -d tag_name
 查看所有标签 git tag 
 搜索标签 git tag -l 'xxx*'
 
 使用 git show '标签' 来查看标签的详细信息
 
 
 远程标签
 
 git push origin 'tag名称' 'tag名称2' # 推送到远程仓库
 git push origin --tags # 将本地尚未推动到远程的标签全部推送到远程仓库
 完整的推送语法
 git push origin refs/taga/v7.0:refs/tags/v7.0
 
 git pull # 会把所有的标签同步到本地
 git fetch origin tag v7.0 # 只会把v7.0标签的信息拉到本地
 
 git push origin :refs/tags/v6.0 #删除远程仓库中的标签
 新版本预发 git push origin --delete tag v5.0
```

git blame '文件名' # 用来查看文件的最近的修改信息

### git diff 相关

```
直接运行 git diff 会比较工作区与暂存区之间的差异(源文件是暂存区，目标文件是工作区)
git diff 'commit id' 比较工作区与某个提交之间的差别，最新的用HEAD(源文件是提交，目标文件是工作区)
git diff --cached 比较的是最新的提交与暂存区之间的差别（源文件是提交，目标文件是暂存区）
git diff 'branch1' 'branch2' 比较两个分支之间的差异。

```

### git 远程相关

```
git remote add origin http://github.com/xxx # 其中的 origin是可以任意的起名字的，但是一般不会改
推送的时候 git push -u origin master 推送一个远端的master，后面可以直接调用git push #完整命令是 git push origin src:dest
对于上一条命令，新版本推荐使用：git push --set-upstream origin develop # 用来在远程分支不存在的时候，直接在远程创建分支 ，想要名字不一样，也可以使用完整的命令 git push --set-upstream origin develop:develop1。
当upstream和本地分支的名字不一样的时候，在push的时候有两种选择：
	git push origin HEAD:develop1 # 推送到upstream 等价于 git push origin develop:develop1
	git push origin develop # 推送到同名分支

git remote show 查看所有关联的远程仓库
git remote show '远程别名' 来查看详细信息

git 的push有两种模式，matching 和 simple， matching会推送到本地名和远程名一样的分支中，而simple会推送到pull的那个分支上。

-------git配置远程ssh链接
(1)ssh-keygen
(2)将共钥放到github中的公钥中

本地			本地						远程仓库
master    origin/master  master

git clone '远程地址' 将远程的项目客隆到本地
git clone '远程地址' ’本地文件名' # 不使用git的远程仓库的名称作为命名

git pull = git fetch + git merge (merge远程分支到本地分支)
git pull -r 可以直接rebase，不会有多有的merge commit 被推送到远程仓库
git pull 的完整命令 git pull origin src:dest (src是远程分支，dest是本地分支)
git fetch origin master:refs/remotes/origin/mymaster


删除远程分支 git push origin :dest #也就是将一个空分支推送到远程仓库。
           git push origin --delete dest # 新版本的git所提供的命令

重命名远程分支 只能先删除远程分支，然后重新push一个新的分支。

git remote prune origin # 在远程仓库中删除了某个分支以后，本地的远程分支会出现不同的情况，可以使用改命令来剪除多余的分支信息。


remote refs
在缺省情况下，refspec会被git remote add命令所自动生成，git会获取远端上refs/heads下的所有引用，并将它们写到本地refs/remotes/origin目录下，所以如果远端上有一个master分支，在本地就可以通过下面几种方式来访问它们的历史记录：
git log origin/master
git log remotes/origin/master
git log refs/remotes/origin/master

git remote rm origin # 删除远程仓库


```

git config --global alias.br branch # git可以给常用的命令起别名。

git config --global alias.unstage 'reset HEAD' # 可以给复杂命令起别名。

git config --global alias.ui '!gitk' # 想要执行单独的命令的时候前面见一个! ， 前面就不会拼接git了。

git gc 命令    会把refs 目录下的相关文件压缩到 packed-refs文件中

​						会压缩objects目录下的内容

git init --bare #会创建一个git裸库， 所谓的git裸库也就是没有工作区的git库。

### git submodule

```
git submodule add '远程的地址' ’要放置子模块的目录‘ #配置一个submodule，要放置子模块的文件夹事先不能存在
如果远程的 submodule有任何变化，只需要进入到submodule文件夹中执行一次 git pull就能进行同步了。
在根目录中执行 git submodule foreach git pull 可以拉取所有的submodule

如果在git中使用了submodule那么在进行git clone的时候是不会对submodule中的代码进行clone的，需要手动clone：
git submodule init
git submodule update --recursive
在git clone的时候，加上 --recursive 就会将所有的子模块都clone下来


git submodule移除需要进行一系列操作:
git rm --cached 'submodule'
rm -rf 'submodule'
git commit -am 'xxx'
git push

rm .gitmodules
git add .
git commit -m 'xxx'
git push
```

### git subtree

能够完成submodule能完成的所有功能，同时对于双向修改的支持更好，更完善。

```
//在parent中的操作 
git remote add subtree-origin 'xxxx' # 添加subtree所对应的远程库
// 要使用--squash就一直使用，要不使用就一直不使用
git subtree add --prefix=subtree subtree-origin master [--suqash] # 参数分别问 文件夹 远程库 分支

git subtree pull --prefix=subtree subtree-origin master # 拉取远程分支

git subtree push --prefix=subtree subtree-origin master # 推送到远程分支
```

### cherry-pick

将某个分支的提交应用到另外一个分支上。

```
git cherry-pick 'commit-id' #将某个提交应用到当前分支上

```

### rebase

```
merge
	* ← * ← * ← * dev
		↖ * ← *  ↙ test
		
rebase
              dev     test
	* ← * ← * ← * ← * ← *
	
rebase过程中也会出现冲突
解决冲突后，使用git add添加，然后执行 git rebase --continue
接下来git会继续应用余下的补丁
任何时候都可以哦谈过如下命令终止rebase，分支会恢复到rebase开始前的状态 git rebase --abort

git rebase --skip # 在发生冲突的时候，直接应用前一个的内容，当前的点就跳过了。
rebase最佳实践
- 不要对master分支执行rebase，否则会引起很多问题。
- 一般来说，执行rebase的分支都是自己的本地分支，没有推动到远程版本仓库。

```

## gradle

```
插件
apply plugin: 'java'
apply plugin: 'war'
apply plugin: 'idea'
apply plugin: 'maven'

申明自己
group 'com.test'
version '1.0'

gradle 中的依赖
	compile 'g:a:v' 或者 group: 'g',name: 'a', version: 'v'
	
	# testCompile providedCompile
	# gretty 是一个很好用的插件，可以直接代替tomcat	

配置字符集
tasks.withType(JavaCompile) {
	options.encoding = 'UTF-8'
}
[compileJava, javadoc, compileTestJava]*.options*.encoding = 'UTF-8'


```

