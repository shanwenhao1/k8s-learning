# Helm基本使用
[文档](https://www.qikqiak.com/k8s-book/docs/43.Helm基本使用.html)

## 仓库
Helm的Repo仓库和Docker Registry比较相似, Chart库可以用来存储和共享打包 Chart 的位置, 安装Helm后, 默认仓库地址是
google的一个地址, 无法科学上网的无法访问到官方提供的Chart仓库．使用`helm repo list`查看当前的仓库配置.

Chart 仓库其实就是一个带有`index.yaml`索引文件和任意个打包的Chart的HTTP服务器而已

