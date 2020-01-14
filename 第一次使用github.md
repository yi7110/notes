**配置全局变量**  
`git config --global user.name "liuzf"`  
`git config --global user.email "592725127@qq.com"`  
**配置ssh**  
`cd ~/.ssh`  
`ssh-keygen -t rsa -C "592725127@qq.com"`  
**生成id_rsa 私钥 id_rsa.pub公钥，将公钥添加到github的setting:**  
就可以通过ssh访问  
可以通过命令：  
`ssh -T git@github.com`
提示`You are successfully` 意味着配置成功  

**然后找到本地文件夹执行下列命令，完成上传：**  
`git init` 初始化文件夹  
`git add` 文件名  
`git commit -m "说明"`  
`git remote add origin url` 这里的url可以是ssh或者https的类型origin只是一个标识符，会在.git/config文件里生成配置

`git push -u origin master` 可能会失败，需要先pull下来文件  
`git pull origin master` 失败的话，改用  
`git pull --rebase origin master`  
然后push即可；第一次添加-u参数是为了跟远程库建立关联，以后就不需要在加-u参数  

**扩展知识：**  
`git pull = git fetch + git merge FETCH_HEAD`  
`git pull --rebase = get fetch + git rebase FETCH_HEAD`  


