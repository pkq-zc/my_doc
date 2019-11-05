# centos7 搭建gitosis

## 服务器安装软件

1. 使用root或者sudo执行```yum install -y python python-setuptools git-core```
2. 下载gitosis项目:```git clone https://github.com/tv42/gitosis.git```
3. 安装进入刚刚克隆下来的gitosis项目,安装:```python setup.py install```

## 本地客户端电脑公钥上传

1. windows电脑:进入```C:\Users\XXX(用户名)\.ssh```目录下看有没有```id_rsa.pub```文件.如果没有执行```ssh-keygen -t rsa```生成.如果是linux,进入```~/.ssh/```目录,查看是否有该文件,如果没有,使用同样方式生成.
2. 上传公钥到服务器```/tmp```目录下

## 服务器初始化gitosis

1. 新建用户```useradd -m git```
2. 切换用户```su - git```
3. 初始化gitosis```gitosis-init</tmp/id_rsa.pub```.注意```id_rsa.pub```文件为之前客户端创建的公钥.
4. 进入```/home/git/```目录,查看是否存在```gitosis```和```repositories```文件夹.

## 客户端克隆gitosis-admin

1. 客户端执行```git clone git@服务器地址:gitosis-admin.git```
2. 进入```gitosis-admin```查看,存在一个配置文件```gitosis.conf```和一个目录```keydir```文件夹.查看```gitosis.conf```文件.
3. 添加新用户时,将用户的公钥上传至```keydir```目录.最好以```用户名.pud```形式命名.

## 本地新建项目上传

进入```gitosis-admin```,修改```gitosis.conf```如下:

```
[gitosis]

[group gitosis-admin]
members = tom
writable = gitosis-admin

[group test]
members = tom jack # 添加 tom 和 jack 两个用户
writable = test # 项目名称
```

将改动提交到服务器:

```bash
mkdir test # 创建目录
cd test
echo 'this is a test txt' > readme.md # 创建测试文件
git init # git初始化
git add . # 添加修改
git commit -m '初始化提交'
git remote add origin git@主机地址:test.git # 关联远程仓库
git push origin master # 提交到远程仓库
```

登入远程服务器,查看```/home/git/repositories```是否存在推送上来的新项目
