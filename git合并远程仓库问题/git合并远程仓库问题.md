# 记一次git合并远程仓库问题

## 问题描述：

在github上添加了一个新的仓库并且增加了README.md文件后，clone到本地进行环境搭建，此时使用IDEA的`New -> Module`添加一个新的模块，此时直接在当前目录下有新建了一个文件夹以及一系列文件，这并不是我想要的目录结构，希望新的项目结构和README.md文件平级。重新拷贝Module下的工程结构出来后，IDEA又无法识别出来Maven。

所以决定先新建一个项目，再将远程仓库合并进来进行关联。



## 步骤如下：

1. 新建Spring Boot项目，略

2. 点击菜单栏**CVS**，在当前项目下添加git

3. 添加远程目录关联，这里关联的是我自己的目录

   ```bash
   git remote add origin https://github.com/553234736/MenstrualAssistant
   ```

4. 拉取远程仓库代码，此步骤的额外参数非常重要，否则会导致后续无法推送的远程仓库

   ```bash
   git pull origin master --allow-unrelated-histories
   ```

5. 处理可能出现的冲突，没有当然是最好的啦～如果出现需要合并的冲突，解决完无法使用IDEA的**commit**工具，提示需要合并。此时在命令后输入查看待解决的文件。

   ```bash
   git status
   ```

   一般情况下可以直接执行将文件进行add即可

   ```bash
   git add .
   ```

   注：add后面的"**.**"表示添加所有文件，记得将不需要进行版本控制的文件及文件夹写到`.gitignore`文件中即可。



## 参考文档：

https://blog.csdn.net/mygodit/article/details/84869504

https://www.liaoxuefeng.com/wiki/896043488029600/898732864121440

