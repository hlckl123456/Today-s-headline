# Development Report

## 1.Project summary

### 1.1 Brief intro

A dynamic website which collect, organize latest news with personalized recommendations.

Server side:

- **Node.js** is used as server side 

- **MongoDB** is used for data storage.

- **Redis** is used for cache query.
- Algorithm and datasets are prepared for personalized recommendations.
- In user management, we also use mailbox to do authentication register.

Data support:

- Python BeautifulSoup is used to crawl and parse news pages of popular sites in China.
- For news data processing, we use the datasets from Wikipedia and use Word2Vec algorithm and relevant tools to establish chinese keywords modeling. We use this model to do recommend the news that user interested according to relavant words.
- We also use Microsoft Axure’s recommendation api to implement personalized recommendation.

### 1.2 Modules

- News data module:
  - News crawler module: crawl webpage from popular sites and store into databse
  - News cluster module: establish keyword model based on chinese wiki corpus clustering
  - User recommendatio model: use the RESTful API provided by Microsoft Axure
- Web module
  - front side
  - Server side
- Android module

### 1.3 Source code structure

NewsFeeds
├── **Graphs**  system architecture
├── **README.md** Development document
├── **NewsFeed_Android** Android Source code
├── **Scripts** News data related script
│   ├── **word2vec** Chinese corpus training script
│   ├── **config.ini** Config file
│   ├── **modelUpdater.py** Recommended system related script
│   ├── **newsCrawler.py** News crawl script
│   ├── **service.py** Server background resident process script
└── **Server** 
├── **app.js Node.js** 
├── **bin** Server startup script
├── **config.json** Website configuration file (server address, database source, etc.)
├── **package.json Node.js** Dependence declaration\
├── **public**  static website file
├── **routes** Node.js routes
├── **scripts** Website backend logic implementation
└── **views** front end

### 1.4 Running environment

- Operating system: Windows/Linux/OSX
- Server software: Node.js 6.9.0 and above
- Logical database: MongoDB 2.4.1 and above
- Cache database: Redis 3.2.9 and above

### 1.5 User instructions

1.Enter the directory where the **Server** is located. 

2.According to the individual server configuration, change the configuration file **config.json** as follows：

```json
{
  "host": "服务器主机地址",
  "port": "服务器主机端口",
  "connect": "MongoDB连接字符串",
  "newscol": "MongoDB新闻数据库名",
  "usercol": "MongoDB用户数据库名",
  "mailoptions": {
    "service": "邮件服务",
    "email": "邮箱账号",
    "password": "邮箱密码"
  },
  "redis": {
    "port": "Redis端口",
    "host": "Redis服务器地址"
  },
  "database": {
    "host": "MongoDB数据库服务器地址",
    "port": "MongoDB服务器端口",
    "user": "Mongo用户名",
    "password": "密码"
  },
  "api": {
    "modelid": "微软认知服务-推荐系统模型ID",
    "endpoint": "https://westus.api.cognitive.microsoft.com/recommendations/v4.0",
    "token": "微软认知服务认证Token"
  },
  "genres": {
    "personal": "推荐",
    "hot": "热门",
    "society": "社会",
    "domestic": "国内",
    "global": "国际",
    "technology": "科技",
    "finance": "经济",
    "war": "军事",
    "education": "教育",
    "car": "汽车",
    "game": "游戏",
    "discover": "探索",
    "entertain": "娱乐",
    "fashion": "时尚",
    "health": "健康",
    "history": "历史",
    "mobile": "数码",
    "sport": "体育",
    "travel": "旅游"
  }
}
```

3.Install the Node.js dependency package：

```sh
npm install
```

4.Start the server

```sh
npm start
```

## 2.News crawl module

### 2.1 database design

Since this project uses MongDB based on document storage, each news uses a JSON-like data type store and does not require a fixed schema (Schema). However, in order to facilitate subsequent modeling and display, the news crawling module needs to store the data crawled by each news website in the database according to the following fields and their corresponding meanings:

```json
{
	"_id" : 新闻id,
	"title" : 新闻标题（唯一索引项）,
	"source" : 新闻来源,
	"time" : 发表时间,
	"abstract" : 新闻摘要,
	"comments_count" : 新闻评论数,
	"favorite_count" : 新闻收藏数,
	"genre" : 新闻类别,
	"has_image" : 是否有图片,
	"imgurls" : 图片链接(数组),
	"keywords" : 标题包含关键词(数组), 
	"related_words" : 其他相关关键词(数组),
	"uploaded" : 是否已上传到微软认知服务模型
}
```

### 2.2 News crawl module

In this project, the news crawling module mainly realizes the crawling of news data of tencent, netease, ifeng.com, toutiao and other four websites. For each story parsed from each site, the fields and formats that end up in the database need to be consistent as described in the previous section.

Since all you get from the site is HTML data (apart from the fact that toutiao uses the API to get the json-formatted news data), you need to use an HTML parser like BeautifulSoup to get the items that make sense in the HTML DOM. To climb tencent new script implementation as an example:

Firstly, find out the webpage link of each type of tencent news and store it in a Dict object:

```python
self.ADDRS_LIST = {
    'http://society.qq.com/': 'society',
    'http://ent.qq.com/': 'entertain',
    'http://sports.qq.com/': 'sport',
    'http://finance.qq.com/': 'finance',
    'http://mil.qq.com/mil_index.htm': 'war',
    'http://news.qq.com/world_index.shtml': 'global',
    'http://cul.qq.com/': 'culture'
    ...
}
```

After looking at the source code of the web page, it can be found that the DOM tree structure of each kind of news website is similar, and each news in the news list is contained in a q-plist element: