## hexo + github 搭建博客

### 安装hexo

npm install -g hexo-cli

hexo init blog

cd blog

hexo server

127.0.0.1:4000

hexo new page '关于我'

### 启动

hexo server

### 新建

hexo new 'XXXXX'(博客标题)

### 提交
 
 hexo generate --deploy
 
 
 
 ---
 title: Hello World
 ---
 Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).
 
 ## Quick Start
 
 ### Create a new post
 
 ``` bash
 $ hexo new "My New Post"
 ```
 
 More info: [Writing](https://hexo.io/docs/writing.html)
 
 ### Run server
 
 ``` bash
 $ hexo server
 ```
 
 More info: [Server](https://hexo.io/docs/server.html)
 
 ### Generate static files
 
 ``` bash
 $ hexo generate
 ```
 
 More info: [Generating](https://hexo.io/docs/generating.html)
 
 ### Deploy to remote sites
 
 ``` bash
 $ hexo deploy
 ```
 
 More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)
