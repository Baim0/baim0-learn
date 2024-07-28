## 下载ollama

整个过程需要准备三个软件：

Ollama。用于运行本地大模型。如果使用闭源大模型的API，则不需要安装Ollama。

Docker。用于运行AnythingLLM。

AnythingLLM。知识库运行平台，提供知识库构建及运行的功能。

1 安装Ollama
下载Ollama（网址：[https://ollama.com/download]()）


下载后直接安装，然后启动命令行窗口输入命令加载模型。命令可以通过点击官网Models后，搜索并选择所需要的模型后查看。


搜索框输入qwen


选择模型后，拷贝对应的命令
例如: 
```bash
ollama run qwen2
```

这样就能本地跑千问了。

注：Ollama支持加载运行GGUF格式的大模型，这个自行查看官网。

启动命令行窗口，拷贝命令并运行，若是第一次运行，Ollama会自动下载模型并启动模型。如本机上已安装了qwen:14b模型，则输入命令后会直接启动此模型。

![](../img/Pasted%20image%2020240728130237.png)

## web ui

然后呢，大部分人是用了open-ui，但是我觉得没必要，这里推荐一个谷歌宝藏插件：Page Assist

![](../img/Pasted%20image%2020240728130417.png)
