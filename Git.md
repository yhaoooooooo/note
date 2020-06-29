1. git status 查看工作区代码相对于暂存区的差别
2. git add . 将当前目录下修改的所有代码从工作区添加到暂存区 . 代表当前目录
3. git commit -m ‘注释’ 将缓存区内容添加到本地仓库
4. git pull origin master先将远程仓库master中的信息同步到本地仓库master中
5. git push origin master 将本地版本库推送到远程服务器，
    origin是远程主机，master表示是远程服务器上的master分支和本地分支重名的简写，分支名是可以修改的