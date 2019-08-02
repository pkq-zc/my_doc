# git commit 撤销操作

如果执行完```git commit```之后,还有文件未提交该怎么办呢?

- 添加忘记提交的文件到暂存区```git add 忘记提交的文件```
- 执行```git commit --amend```,这个时候会弹出提交信息说明,编辑说明后保存后,这一次的提交会和上一次提交在```git log```中只会显示一条记录.

```bash
zengchao@DESKTOP-7MTSEAF MINGW64 ~/Desktop/test (master)
$ touch {1,2}.txt

zengchao@DESKTOP-7MTSEAF MINGW64 ~/Desktop/test (master)
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        1.txt
        2.txt

nothing added to commit but untracked files present (use "git add" to track)

zengchao@DESKTOP-7MTSEAF MINGW64 ~/Desktop/test (master)
$ git add 1.txt

zengchao@DESKTOP-7MTSEAF MINGW64 ~/Desktop/test (master)
$ git commit -m 'first commit'
[master ca775ff] first commit
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 1.txt

zengchao@DESKTOP-7MTSEAF MINGW64 ~/Desktop/test (master)
$ git add 2.txt

zengchao@DESKTOP-7MTSEAF MINGW64 ~/Desktop/test (master)
$ git status
On branch master
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)

Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        new file:   2.txt


zengchao@DESKTOP-7MTSEAF MINGW64 ~/Desktop/test (master)
$ git commit --amend
[master 70055fe] first commit,second commit
 Date: Fri Aug 2 22:53:06 2019 +0800
 2 files changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 1.txt
 create mode 100644 2.txt

zengchao@DESKTOP-7MTSEAF MINGW64 ~/Desktop/test (master)
$ git log
commit 70055feb15568cd59976c2bf660babb8e8cecec8 (HEAD -> master)
Author: zengchao <zengchaowork@foxmail.com>
Date:   Fri Aug 2 22:53:06 2019 +0800

    first commit,second commit

```