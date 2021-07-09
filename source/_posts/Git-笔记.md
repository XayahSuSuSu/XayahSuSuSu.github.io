---
title: Git 笔记
date: 2021-07-09 00:18:18
tags: [ 'Git', 'GitHub', '笔记' ]
---
# [Git-Notes（GitHub地址）](https://github.com/XayahSuSuSu/Git-Notes)

1. 删除Commit(删除当前分支最新的一条Commit):

	```
	1) 本地删除Commit记录
	git reset --hard HEAD^

	2) 向远程仓库提交强制申请
	git push origin HEAD -f
	```

2. 从其他远程仓库导入项目
	
	```
	1) 首先将导入仓库克隆到本地
	git clone <gitUrl(导入仓库)>

	2) cd 到该项目根目录

	3) 将该项目强制导入到被导入项目
	git push --mirror <gitUrl(被导入仓库)>
	```

3. 导入其他仓库的Commit记录

	```
	1) cd 到被导入项目根目录

	2) 将该库加为远程仓库
	git remote add target <gitUrl(导入仓库)>

	3) 将远程代码抓取到本地
	git fetch target

	4) 使用cherry-pick命令提交转移
	git cherry-pick <commitHash>
	```

4. 修改最新的一条Commit记录信息

	```
	1) cd 到被导入项目根目录

	2) git commit --amend

	3) 输入i进入插入模式

	4) 修改Commit记录信息

	5) 按Esc退出

	6) 输入:wq保存

	7) git push -f
	```

5. 导入其他仓库的分支

	```
	1) cd 到被导入项目根目录

	2) 将该库加为远程仓库
	git remote add target <gitUrl(导入仓库)>

	3) 将远程代码抓取到本地
	git fetch target

	4) 切换到远程仓库对应分支
	git checkout <对应分支名称>

	5) 基于该分支创建本地分支
	git checkout -b <本地分支名称>

	6) 推送当前分支并建立与远程上游的跟踪
	git push --set-upstream origin <本地分支名称>
	```

6. 修改历史Commit记录

	```
	1) 列出历史Commit列表(n为项数)
	git rebase -i HEAD~n

	2) 输入i进入插入模式

	3) 将被修改Commit前面的 "pick" 修改为 "edit"

	4) 按Esc退出

	5) 输入:wq保存

	6) git commit --amend

	7) 输入i进入插入模式

	8) 修改Commit记录信息

	9) 按Esc退出

	10) 输入:wq保存

	11) git rebase --continue

	12) git push -f
	```

7. 修改历史Commit提交时间

	```
	1) 列出历史Commit列表(n为项数)
	git rebase -i HEAD~n

	2) 输入i进入插入模式

	3) 将被修改Commit前面的 "pick" 修改为 "edit"

	4) 按Esc退出

	5) 输入:wq保存

	6) GIT_COMMITTER_DATE="2021-04-29T11:00:00" git commit --amend --date="2021-04-29T11:00:00"

	7) 输入:wq保存

	8) git rebase --continue

	9) git push -f
	```

8. 撤回上次Commit并保留修改的文件(在Commit时恢复上次Commit信息):

	```
	1) 撤回上次Commit
	git reset --soft HEAD^

	2) Commit并恢复上次Commit信息
	git commit -C HEAD@{1}

	3) 向远程仓库提交强制申请
	git push origin HEAD -f
	```