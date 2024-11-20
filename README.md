# 概念解析    
部署方案是基于fastgpt(https://doc.tryfastgpt.ai/docs/development/docker/)的知识库管理系统，以oneapi为媒介调用ollama管理的大语言模型，具体的实现架构如下图所示。
![图片alt](https://cdn.jsdelivr.net/gh/yangchuansheng/fastgpt-imgs@main/imgs/sealos-fastgpt.webp '使用docker部署fastgpt的技术方案')
1. docker: 类似于一个包管理软件，打包所有需要的系统环境或软件内容，放在本地以后即可离线访问不同的组件。项目使用docker技术将这些工具打包在一起统一配置，但是拉取docker镜像（image）的时候可能需要科学上网，全程开启clash的tun mode。后续如果还想进一步学习使用，可以在docker hub上找对应的镜像。https://hub.docker.com/

2. oneapi: 一种将大模型交互行为统一为openAI接口格式的工具，便于系统管理，需要配置令牌、模型渠道等等，后续在此基础上在线发布或收费啥的都比较方便。

3. ollama: 著名的开源大模型分发库，可以很方便的下载大模型进行命令行式的调用，是本项目的基建。

4. open webui: 在ollama的基础上，制作的网页访问界面，调用ollama进行类chatgpt式的访问，用docker部署运行后可以将ollama运行在特定的11434端口，便于oneapi调用。如果不需要构建知识库，只想体验本地大模型，其实用ollama和open webui就能实现。

5. postgreSQL: 一个向量数据库，其中pg代表向量，SQL在此方案中指代mysql，后续会在配置docker-compose.yml时体现。因为大模型无法直接处理知识库中各种文本类型的数据，所以需要预处理与嵌入模型将文本编码为稠密连接的数学向量。postgreSQL就是用来存储这些结构化向量的数据库。

6. MongoDB: 也是一个数据库，用于存放你准备的私域数据，如pdf，excel，txt等，实际使用时是用docker一键拉取并运行的，无需了解其内部的具体操作方式。

7. m3e: 如前所述，是一个开源的向量模型，用于对文本进行嵌入，生成向量。你也可以替换成openAI的嵌入模型，或者更适用于中文的bge模型，我还没试过，但是部署方式应该类似。

8. fastgpt: 将知识库与大模型连接的核心，也是我们使用时要登入的界面。oneapi承担了一个运营商要做的部分，fastgpt即承担用户需要的部分，让你可以导入知识库、访问各种部署好的模型、设定提示词、构建应用等等。
# 正式开始部署
## 安装docker
1. 可以使用命令行安装，也可以下载桌面软件，建议下载一个桌面软件，看运行日志比较方便，后面还会有各种坑要解决，不看日志不行。

2. docker安装后需要添加到环境变量，否则系统终端可能会无法识别命令。记得也要把原始的环境变量添加到/etc/profile里，否则会覆盖并丢失原生的系统命令，如nano, ls, vim.
## 安装ollama
1. 很简单，进网址直接下载就是
https://ollama.com/download

2. https://ollama.com/search在这个网址可以搜一下你想要的大模型，然后在命令行下载，如    
    ~~~
    ollama run qwen2.5-coder
    ~~~
    运行后即可下载这个千问模型到本地了。

2. 安装好以后，可以在命令行调用ollama进行大模型对话。
    ~~~
    ollama run qwen2.5-coder
    ~~~
    想退出的话按ctrl+D即可。

至此就完成了ollama的部署，已经初步可以使用大模型了，但是命令行的形式还是不够方便，所以我们需要安装一个open webui。

## open webui
1. 该项目的github链接为https://github.com/open-webui/open-webui。
参考项目主页的介绍，由于我们已经有了ollama在本机，所以使用如下的代码安装：
~~~
docker run -d -p 18080:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
~~~
2. 在命令行输入以上代码，即可完成open webui的安装，如果一直超时报错，那说明你需要科学上网。**注意：项目界面给的命令端口号选项是3000:8080， 意味着使用本机的3000端口，和后面oneapi的部署冲突了，所以我改成了18080端口。你可以直接复制我给的命令。**
3. 完成安装后，docker会自动运行，此时可以打开浏览器输入地址：
~~~
localhost:18080
~~~
即可进入默认的open webui界面，初次登录需要注册一个账号，默认是超级管理员。不想动脑子的话听我的：
~~~
用户名：root
密码：1234
邮箱：xx@zju.edu.cn
~~~
4. 注册后登录即可进行大模型对话了，选择模型后对话，和gpt用法一样。
## fastgpt
接下来就是本项目的重头戏，构建知识库系统，基本上是参考官方文档的设置。(https://doc.tryfastgpt.ai/docs/development/docker/)

1. 在本地新建一个fastgpt的文件夹，并下载config.json和docker-compose.yml到该文件夹下：
~~~
mkdir fastgpt
cd fastgpt
~~~
- 如果你是linux，你可以用官网给的代码下载；
- 如果是mac，就手动下载，复制到这个文件夹下(注意要下载docker-compose-pgvector.yml)；
- 如果是windows，建议你用WSL2模拟一个linux操作，节省点头发比较好。

2. 接下来，则需要对下载好的docker-compose-pgvector.yml进行修改。先把他重命名为docker-compose.yml，这个步骤也许多余，也许不多余，我没测试过。

3. 在文档内部，针对我们的情况，主要是针对我们的情况做一些修改。因为我是m2 pro mac，尝试使用最新版的mongo无法跑通，所以把mongo版本改成了4.4.29
~~~
  mongo:
    image: mongo:4.4.29 # dockerhub
~~~
然后，在‘fastgpt:'的'environment:'里，把openai的网址替换为我们的本地docker地址，因为我们是离线使用的。
~~~
OPENAI_BASE_URL=http://host.docker.internal:3001/v1
~~~

在'services:'下，‘oneapi:’的定义后额外添加一个m3e的配置，让docker去这个地址下载m3e模型，支持我们的向量化。注意：这里的参数是默认下载cpu版的m3e模型，gpu版的参数你需要再去镜像网站确认一下。
~~~
  m3e:
    image: stawky/m3e-large-api:latest
    container_name: m3e
    restart: always
    ports:
      - 6008:6008
    networks:
      - fastgpt
~~~

5. 接下来则需要对config.json做一下修改，添加我们的本地模型参数和m3e模型参数。

- 在'"llmModels": '内添加你下载好的ollama模型，注意名字要一模一样对应起来，删掉它原本的chatgpt模型名称，不删的话后面可能会提示找不到gpt4o-mini模型，然后报错。我本地只有一个"qwen2.5-coder:7b"模型，所以我的定义如下：
~~~
 "llmModels": [
  
    {
      "model": "qwen2.5-coder:7b",
      "name": "qwen2.5-coder:7b",
      "avatar": "/imgs/model/openai.svg",
      "maxContext": 125000,
      "maxResponse": 32000,
      "quoteMaxToken": 120000,
      "maxTemperature": 1.2,
      "charsPointsPrice": 0,
      "censor": false,
      "vision": false,
      "datasetProcess": true,
      "usedInClassify": true,
      "usedInExtractFields": true,
      "usedInToolCall": true,
      "usedInQueryExtension": true,
      "toolChoice": false,
      "functionCall": false,
      "customCQPrompt": "",
      "customExtractPrompt": "",
      "defaultSystemChatPrompt": "",
      "defaultConfig": {
        "temperature": 1,
        "stream": false
      },
      "fieldMap": {
        "max_tokens": "max_completion_tokens"
      }
    }
  ],
~~~



- 在'"vectorModels": '里添加一个m3e的内容，模仿其他向量模型的格式即可。
~~~
    {
      "model": "m3e", // 模型名（与OneAPI对应）
      "name": "M3E", // 模型展示名
      "Price": 0, // n积分/1k token
      "defaultToken": 500, // 默认文本分割时候的 token
      "maxToken": 1800 // 最大 token
    },
~~~

## 批量拉取并运行docker
在fastgpt文件夹下，运行
~~~
docker compose up -d
~~~
即可针对我们刚才修改的配置，拉取并运行所有的环境，如果无法运行，可尝试：
~~~
docker-compose up -d
~~~
docker-compose 和docker compose功能几乎相同，有一个能用就行了。这里的up指的是打开， -d指的是在后台运行。

初次运行可能会需要下载很多东西，要等待很久，需要科学上网。拉取成功后，就可以进入图形界面进行配置了。

## oneapi的登录
1. 在浏览器输入
~~~
localhost:3001
~~~
进入oneapi的管理界面，初次登录的默认账号应该是
~~~
root
123456
~~~
密码不对的话试试'1234'或者 'fastgpt', 反正就这么几个。

2. oneapi承担了后台的功能，需要进行令牌(token)、计费、渠道(模型来源)等功能。因为我们用的是本地离线模型，所以这个收费只是写写的，不会真的收钱，放心使用。接下来我们做一些必要的配置。
- 创建令牌： 自己添加令牌设置永不过期、无线额度，然后复制令牌的token，覆盖到前面的docker-compose.yml中的fastgpt下的environment下的CHAT_API_KEY里，注意下面的只是示例，必须改成你自己复制的token。
~~~
      # AI模型的API Key。（这里默认填写了OneAPI的快速默认key，测试通后，务必及时修改）
      - CHAT_API_KEY=sk-ih89Z5qs7A3oQV6U9b7cDcA26aD447Be94499aD3C600B8B5
~~~~

- 添加渠道: 分别添加m3e和ollama的渠道。
    - 对于m3e来说，类型选择openAI, 名称为m3e,模型清除所有模型后，自定义为m3e，秘钥为sk-aaabbbcccdddeeefffggghhhiiijjjkkk，代理为http://host.docker.internal:6008
    - 对于ollama渠道来说，类型输入’自定义渠道‘，BaseURL为http://host.docker.internal:11434， 名称为ollama，模型填入你ollama本地下载的所有模型名称，注意不要填错，秘钥填'sk-key'。    
- 运营设置: 因为我们这时是一个后台，所以理论上需要对各种模型计费，默认的oneapi是没有对m3e模型计费比率的设置的，所以我们进入设置-运营设置-倍率设置-模型倍率，添加一个m3e的倍率：
~~~
  "m3e": 1,
~~~
保存就行了。不用担心，这里涉及到钱的内容对我们来说只是数字，我们是无限额度用户☺️。

## fastgpt的登录
1. 登录fastgpt之前， 为了让我们之前的更改生效，需要重新运行一遍项目，所以fastgpt文件夹下，命令行再次输入：
~~~
docker compose down 
~~~
等待全部关闭后，输入:
~~~
docker compose up -d
~~~

2. 运行后，可以尝试登录fastgpt，在浏览器输入：
~~~
localhost:3000
~~~
默认登录账号是：
~~~
root
1234
~~~
登录以后即可创建自己的知识库。

# 构建知识库并使用
本部分比较简单，可以自己尝试。
知识库-新建-通用知识库，向量化模型选择m3e， 然后导入文本数据集，选择分块和索引方式，等待它后台自动创建完毕，就可以在模型中引入知识库了。
- 如果使用中遇到问题，可以在docker- desktop程序中查看各种容器的实时运行日志。尝试百度或者问问gpt解决问题吧。

# 比较容易踩的坑
1. 没有科学上网
2. 浏览器代理未关闭，导致容器之间的网络通信存在问题。clash全程开tun模式没问题，就是比较费流量。
3. mongo的版本号不匹配
4. 渠道的代理地址没写对，如冒号变点号，端口没有写对等等（要和你的yml文件完全对应）。建议查bug的时候先测试一下oneapi的两个渠道能否通信成功，再找mongo和fastgpt的问题。
5. cpu版本构建索引慢是正常的，不要觉得卡死了，我两个pdf文档也构建了很久才能用， 对话也要响应很久。 docker日志里的通信都是200 ok 那就问题不大，耐心等待即可。

# 作者信息
Zeyu Cao: [jjwwczy](https://github.com/jjwwczy)   
参考视频:    
https://www.bilibili.com/video/BV1cxS4YyER2/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=40039378df3c387f373717f5153ba3af  
参考文档: https://doc.tryfastgpt.ai/docs/development/docker/  
撰写日期:   2024.11.20
