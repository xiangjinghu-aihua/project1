# git基本命令

@xiangjinghu-aihua

## git init
*作用：把普通文件夹变成 Git 可以管理的仓库。*
'''cpp
# 1. 先进入你的项目文件夹
cd 你的项目路径

# 2. 初始化 Git 仓库
git init
'''

## git add
*作用： 把文件加入 “暂存区”*
'''cpp
# 添加单个文件
git add 文件名

# 添加所有修改/新增文件（最常用）
git add .
'''

## git commmit
*生成一个版本快照，永久保存当前代码状态。*
'''cpp
git commit -m "这里写本次修改的说明"
'''

## git push
*把你本地的代码上传到云端（GitHub/Gitee）*
'''cpp
git push 远程仓库名 分支名

# 最常用默认写法
git push origin main
'''