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

