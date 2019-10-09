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
  "host": "Server host",
  "port": "Server host port",
  "connect": "MongoDB connection string",
  "newscol": "MongoDB news database name",
  "usercol": "MongoDB user database name",
  "mailoptions": {
    "service": "Mail service",
    "email": "Mail account",
    "password": "Mail password"
  },
  "redis": {
    "port": "Redis port",
    "host": "Redis host address"
  },
  "database": {
    "host": "MongoDB host address",
    "port": "MongoDB port",
    "user": "Mongo username",
    "password": "password"
  },
  "api": {
    "modelid": "Microsoft cognitive services-recommendation system modelID",
    "endpoint": "https://westus.api.cognitive.microsoft.com/recommendations/v4.0",
    "token": "Microsoft cognitive services certification token"
  },
  "genres": {
    "personal": "recommended",
    "hot": 
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

```
{
  "_id" : news id,
  "title" : news headline (unique index item),
  "source" : news source,
  "time" : the time of publication,
  "abstract" : news summary,
  "comments_count" : number of news comments,
  "favorite_count" : number of news collections,
  "genre" : news category,
  "has_image" : Is there a picture,
  "imgurls" : image link (array),
  "keywords" : The title contains keywords (array),
  "related_words" : Other related keywords (array),
  "uploaded" : Whether it has been uploaded to the Microsoft Cognitive Service Model
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

Therefore, when parsing HTML, it is first necessary to convert the HTML string into a DOM tree structure through `BeautifulSoup(response.text, "html.parser")`, and then obtain the DOM elements contained in each q-plist by `list.findAll('div', {'class' : ['Q-pList'] })` method, and then use the same method to obtain its internal title, keyword and other contents and store them in the database:

```python
for (addr, catogory) in self.ADDRS_LIST.items():
    response = requests.get(addr, HEADERS)
    if response.status_code != 200:
        print('response code = ', response.status_code)
        exit()
    soup = BeautifulSoup(response.text, "html.parser")
    lists = soup.findAll('div', {'class' : 'list'})
    for list in lists:
        for item in list.findAll('div', {'class' : ['Q-tpWrap', 'Q-pList'] }):
            try:
                linkto = item.find('a', {'class' : 'linkto'})
                title = linkto.text
                docurl = linkto.get('href')
                keywords = item.find('span',{'class':'keywords'}).text
                imgs = [ a.get('src') if a.get('src') else a.get('_src') for a in item.findAll('img')]
                comment = item.find('a', {'class' : 'discuzBtn'})
                if comment:
                    commentNum = comment.text
                    commenturl = comment.get('href')
                source = item.find('span', {'class' : 'from'}).text
                print(catogory, title.strip())
               
            except:
                print_exc()
                exit()

```

### 2.3 news clustering module

The realization idea of the news clustering module is simple. After the keyword library model is established by using Word2Vec, find out the synonyms of each news keyword (keywords field) and record them as the related_words of the news. If the related_words of one news have intersection with the keywords of another news, the two news are considered to be related.

Keywords library model used in the project provided by the Chinese wikipedia [Wiki traditional Chinese corpus](https://dumps.wikimedia.org/zhwiki/latest/zhwiki-latest-pages-articles.xml.bz2) was established, the Scripts are used in the Scripts/word2vec folder. First, we need to convert the downloaded XML file into TXT file, mainly through the script process_wiki.py to execute the command:

```sh
python3 process_wiki.py zhwiki-latest-pages-articles.xml.bz2 wiki.cn.text
```

The whole process takes about 10 minutes. After processing, the text file as follows is obtained:
![image](https://i.imgur.com/RraPydR.jpg)

Since the Chinese corpus of wikipedia only provides the corpus of traditional Chinese, and it can be seen that there are some English and other punctuation characters, we need to convert them into simplified Chinese, divide Chinese words into words, and then remove the useless characters such as English.
Jane traditional transformation mainly through [OpenCC](https://github.com/BYVoid/OpenCC) to implement:

```sh
opencc -i wiki.cn.text -o wiki.cn.text.jian -c t2s.json
```

Chinese word segmentation is not as simple as English has a natural separator, but we can use some word segmentation tools for simple word segmentation. One of relatively good Word segmentation software is [jieba Chinese word segmentation](https://github.com/fxsjy/jieba).
The script seperate_words.py was obtained after partial modification of the given example, and the command was executed:

```sh
python3 separate_words.py wiki.cn.text.jian wiki.cn.text.jian.seq 
```

Write a script remove_words.py, which USES regular expression matching to remove English and punctuation characters, leaving only Chinese words:

```sh
Python3 remove_words. Py wiki. Cn. Text. Jian. The seq wiki. Cn. Text. Jian. Removed
```

Finally, the script train_word2vec_model.py provided by Google word2vec official tutorial is used to train the Chinese word collection that has been processed and conforms to the format requirements:

```sh
Python3 train_word2vec_model. Py wiki. Cn. Text. Jian, removed the wiki. En. Text. Jian. The model wiki. En. Text. Jian. The vector
```

Model files obtained after the training:



![enter image description here](https://i.imgur.com/ZsXlzax.png)

Import model for testing:
![](https://i.imgur.com/Im9cnQ8.png)

![](https://i.imgur.com/LsW7bfH.png)
It can be seen that the trained model can basically meet the requirements of finding synonyms.

After training the model, the next step is to use Word2Vec to find an synonym for each news keyword and save it in the related_words field. In this way, when querying related news of a certain news later, only the intersection of its related_words collection and the keywords collection of other news should be done:

```python
def buildRelated(self):
    collection = getConnection('mongo')[self.colName]
    model = gensim.models.Word2Vec.load(self.word2vecModel)
    try:
        for news in collection.find({ "related_words": { "$exists": 0 } }, projection={"keywords":1, "title": 1, "_id": 1}):
            relatedWords = set()
            for keyword in news["keywords"]:
                try:
                    relatedWords |= set(map(lambda t: t[0], model.most_similar(keyword)))
                except KeyError:
                    print("KeyError:", keyword)
                    continue
            print(news["title"])
            collection.update_one({"_id": news["_id"]}, 
                                    {"$set": { "related_words": list(relatedWords) }})
    except:
        print_exc()
    finally:
        model = None
        gc.collect()
```

Personalized recommendation module mainly use Microsoft in the cognitive service [Suggested API services](https://azure.microsoft.com/zh-cn/services/cognitive-services/recommendations/), according to the specific user's purchase history, recommendation engines use Azure machine learning to build, to provide specific to that user's suggestion, and make them enjoy personalized experience.

The service contains RESTful apis for many different operations, each with very detailed usage documentation. The basic Usage pattern for the service is to first establish a recommendation Model using the Create Model API, then upload all the objects needed for the recommendation (Catalog Item, such as the news in this project) in the specified format of text, and then upload the Usage for each user. Then, the most important Trigger Build API is called to conduct modeling analysis on the data and records that have been uploaded to the library. The following is the description document of the API:
![enter image description here](https://i.imgur.com/Gp5142v.png)

When a Build is done, then to [Get user - to - item recommendations API](https://westus.dev.cognitive.microsoft.com/docs/services/Recommendations.V4.0/operations/56f30d77eda5650db055a3dd) to obtain a user id to specific recommendations for news.

Since the implementation of this part is mostly to construct the request body, send HTTP requests and call RESTful API, the code implementation of each step is much the same, so two examples are used to show the concrete implementation.

When the user clicks on a news, a POST request is sent to upload a Usage data to the API:

```javascript
function uploadUsage() {
    let params = {
        modelId: ""
    }
    let body = {
        "userId": "string",
        "buildId": 0,
        "events": [
            {
                "eventType": "Click",
                "itemId": "string",
                "timestamp": "string",
                "count": 0,
                "unitPrice": 0.0
            }
        ]
    }
    $.ajax({
        url: "https://westus.api.cognitive.microsoft.com/recommendations/v4.0/models/28f64f3a-84a8-4f6d-889b-6c738d284aad/usage/events?"
            + $.param(params),
        beforeSend: function(xhrObj){
            // Request headers
            xhrObj.setRequestHeader("Content-Type","application/json");
            xhrObj.setRequestHeader("Ocp-Apim-Subscription-Key","62155e00332a4a62afbfe6478c8c9212");
        },
        type: "POST",
        // Request body
        data: body,
    }).done(function(data) {
        console.log("update success")
    }).fail(function() {
        alert("update error");
    });
}
```

The back-end process starts statistical modeling once (periodically, such as once a day) using the Trigger Build API:

```python
def triggerBuild(self):
    collection = getConnection('mongo')[self.modelColName]
    header = self.HEADERS_JSON
    buildurl = self.endpoint + '/models/%s/builds?' % (self.modelid)
    body = json.dumps({
        "description": "Simple recomendations build",
        "buildType": "recommendation",
        "buildParameters": {
            "recommendation": {
                "numberOfModelIterations": 40,
                "numberOfModelDimensions": 20,
                "itemCutOffLowerBound": 1,
                "itemCutOffUpperBound": 10,
                "userCutOffLowerBound": 0,
                "userCutOffUpperBound": 0,
                "enableModelingInsights": False,
                "useFeaturesInModel": True,
                "modelingFeatureList": "tag",
                "allowColdItemPlacement": True,
                "enableFeatureCorrelation": True,
                "reasoningFeatureList": "tag",
                "enableU2I": True
            }
        }
    })
    try:
        resp = requests.post(buildurl, body, headers=header)
        result = json.loads(resp.text)
        print("url = ", buildurl)
        print(result)
        if resp.status_code == 202: # success
            collection.insert({
                "buildId": result["buildId"],
                "time": datetime.now().strftime(self.timeFormat),
                "modelId": self.modelid,
                "token": self.token
            })
    except:
        print_exc()
```

### 2.5 background resident process

Since the website needs to provide the latest news in real time, the server backend must run a resident process to periodically crawl the news, update the model, and so on. Here, python's schedule module is mainly used to perform periodic operations, which provides a very humanized interface. For example, when `schedule.every().day.do(updateModel)` is called, updateModel function can be executed every other day from the time of invocation, where day can also be changed into hour, minute and other time units. These interfaces make it easy to implement functions that are called periodically.
The main implementation code of the background resident process script is as follows:

```python
def main():
    global model
    toutiao = newsCrawler.Toutiao()
    tencent = newsCrawler.Tencent()
    netease= newsCrawler.Netease()
    loop = 1
    interval = int(getConfig('default', 'request_interval'))
    schedule.every().hour.do(toutiao.start)
    schedule.every().hour.do(tencent .start)
    schedule.every().hour.do(netease.start)
    schedule.every().hour.do(model.buildRelated)
    schedule.every().day.do(updateModel)
    while True:
        time.sleep(interval)
```

## 3.Backend

### 3.1 Main interface

- Get the news list by category or keyword
  URL：/news/list
  Method：GET
  Format：application/json

  | Parameter |  type  |                         description                          |
  | :-------: | :----: | :----------------------------------------------------------: |
  |   genre   | String | News category. gets the current user's personalized recommendation news when the value is personal |
  |    tag    | String | News keyword. Only take effect when a genre is not specified |
  |   html    |  Int   |                 1 return html, 0 return json                 |
  |   limit   |  Int   |                upper limit of number of news                 |
  |  offset   |  Int   |                            offset                            |

- Gets a news item by news id
  URL：/news/content
  Method：GET
  Format：text/html

  | Parameter |  type  | description |
  | :-------: | :----: | :---------: |
  |    id     | String |   News id   |

- Use the news id to get a news comment
  URL：/news/comment
  Method：GET
  Format：text/html

  | Parameter |  type  | description |
  | :-------: | :----: | :---------: |
  |    id     | String |   News id   |

- Get hot news tags
  URL：/news/tags
  Method：GET

  | Parameter | type | description |
  | --------- | ---- | ----------- |
  |           |      |             |

- Use news id to get the relevant news
  URL：/news/related
  Method：GET
  Format：text/html

  | Parameter |  type  | description |
  | :-------: | :----: | :---------: |
  |    id     | String |   News id   |

- User registration (send authentication email)
  URL：/users/register
  Method：POST
  Format：text/html

  | Parameter |  type  | description |
  | :-------: | :----: | :---------: |
  |   email   | String |    Email    |
  | password  | String |  Password   |
  | fullname  | String |  user name  |

- Log in 
  URL：/users/login
  Method：POST
  Format：text/html

  | Parameter |  type  | description |
  | :-------: | :----: | :---------: |
  |   email   | String |    Email    |
  | password  | String |  Password   |

- User add favorite news
  URL：/users/like
  Method：GET
  Format：text/html

  | Parameter |  type  | description |
  | :-------: | :----: | :---------: |
  |    id     | String |   News id   |

### 3.2 Email authentication

E-mail authentication is mainly to be able to authenticate users registered to use the mailbox is their own legitimate mailbox. In this module, node. js e-verification module is used for verification.

First configure a mailbox to send authentication messages. In order to centralize the configuration information of the server, all the configurations are unified in the config.json file in the root directory of the server:

```json
{
  "host": "123.206.106.195",
  "port": 3000,
  "connect": "mongodb://123.206.106.195:27017/newslist",
  "newscol": "news",
  "usercol": "userdata",
  "mailoptions": {
    "service": "126",
    "email": "newsfeedregister@126.com",
    "password": "newsfeed2017"
  },
  ...
}
```

Email related to the configuration of the unified on mailoptions fields, including email, password is respectively account password, service for the use of email service provider ([the optional service list](https://nodemailer.com/smtp/well-known/)).

Write scripts to configure the e-verification module, including the verification of email title, body format, email activation link format, email address, password, and so on:

```js
nev.configure({
    verificationURL: 'http://' + config["host"] + ':' + config["port"] + '/users/email-verification/${URL}',
    persistentUserModel: User,
    tempUserCollection: 'newsfeed_tempusers',

    transportOptions: {
        service: mailOption["service"],
        auth: {
            user: mailOption["email"],
            pass: mailOption["password"]
        }
    },
    verifyMailOptions: {
        from: 'Do Not Reply ' + mailOption["email"],
        subject: 'Please confirm account for NewsFeed',
        html: 'Click the following link to confirm your account:</p><p>${URL}</p>',
        text: 'Please confirm your account by clicking the following link: ${URL}'
    },

    confirmMailOptions: {
        from: 'Do Not Reply ' + mailOption["email"],
        subject: 'Account register successfully',
        html: '<p>Successful</p>',
        text: 'Your account has been registered successfully'
    },
    hashingFunction: myHasher

}, function (err, options) {
    if( err )
        console.log(err)
})
```

When a user registers, a temporary user needs to be created and saved to the database (the default is 1 day expiration) until the user clicks on the authentication link. Through mongoose, the ORM module of MongoDB, the logical fields of user entities in the database and the encryption method of passwords are defined (bcrypt module is used for encryption, and the algorithm is [Blowfish encryption algorithm](https://zh.wikipedia.org/wiki/Blowfish_(%E5%AF%86%E7%A0%81%E5%AD%A6))).

```js
var userSchema = mongoose.Schema({
    email: String,
    password: String,
    fullname: String,
    interest: Array
});

userSchema.methods.validPassword = function (pw) {
    return bcrypt.compareSync(pw, this.password);
};
```

Finally, when the user successfully clicks the authentication link in the email, the processing logic to turn the temporary user into an official user is implemented:

```js
router.get('/email-verification/:url', function (req, res) {
    let url = req.params.url;

    nev.confirmTempUser(url, function (err, user) {
        if( err ){
            console.log(err)
        }else {
            nev.sendConfirmationEmail(user.email, function (err, info) {
                if( err ){
                    res.render("info", {
                        title: "Notice",
                        message: "Sending confirmation email failed"
                    })
                }else {  // email verification successful
                    // res.cookie('user', new Buffer(user.email).toString('base64'))        // automatically login
                    req.session.user = user
                    res.redirect('/')       // redirect to home page
                }
            })
        }
    })
})
```

### 3.3 User status maintenance

Although cookie is very convenient to use, it has a big disadvantage, that is, all the data in cookie can be modified on the client side, and the data is very easy to be forged. Moreover, if there are too many data fields in cookie, the transmission efficiency will be affected. In order to solve these problems, session is generated, and the data in the session is kept in the server side. Therefore, the website USES session, which is more secure than cookie, to store user state. Node also provides a very convenient express-session module to read and write sessions.

Website access to session is very convenient, only need to configure express-session option in app.js file:

```js
app.use(session({
    secret: randomstring.generate({
        length: 128,
        charset: 'alphabetic'
    }),
    cookie: {
        maxAge: 60000*1000
    },
    resave: true,
    saveUninitialized: true
}));
```

You can then access the session object by accessing the session field in each express request. Note that since the session still needs to store a randomly generated string session-id on the browser side, you need to specify the effective time of the cookie, which is currently set to 100 minutes by default, meaning that the user needs to log in again after 100 minutes.

### 3.4 redis Cache

In order to reduce the request load on the database as the number of users grows, the key-based redis database is used to cache some relatively static data. After adding redis cache, the operation flow of a certain type of query becomes:

```js
Construct the Key by querying the request field
If (query whether the data is in redis cache by Key){
Build the reply package directly from the data in the cache
} else {
Use MongoDB to get the required data
Write the data to the redis cache by Key
Structural recovery package
}
```

First, it is still necessary to configure the host address and service port of redis server in the server configuration file config.json:

```json
"redis": {
  "port": 6379,
  "host": "123.206.106.195"
},
```

Construct the key and query the data in the cache:

```js
let key = JSON.stringify({
    tag: tag,
    genre: genre,
    offset: offset,
    limit: limit
})
redisClient.get(key, function (err, reply) {
    if( err ){
        console.log(err)
        res.json({ message: "redis error"})
    }else if( reply ){
        console.log("redis hit")
        result = JSON.parse(reply);
        renderResult(result)
    }else{
        News.getList(genre, tag, (err, result) => {
            if( err ){
                console.log(err);
                res.json();
            }else{          // success
                cache(key, JSON.stringify(result))
                console.log('result.length = ' + result.length)
                renderResult(result)
            }
        }, offset, limit)
    }
})
```

Write to the cache is required in the processing of multiple requests, so a separate function is encapsulated to write a key value pair to the cache and set expiration according to the parameter:

```js
function cache(key, value, notExpire, time) {
    if( notExpire )
        return redisClient.set(key, value)
    else
        return redisClient.set(key, value, expireFlag, time ? time : expireTime)
}
```

## 4 Test

The effects of the Web and mobile pages are shown in the previous section, which shows the results of related news and personalized recommendations.

### 4.1 Related news test

Related News recommendation result 1 (Related News in the right-hand column)：
![enter image description here](https://i.imgur.com/kmQb3yg.jpg)

Related News recommendation result 2 (Related News in the right-hand column)：
![enter image description here](https://i.imgur.com/2yfBTn6.png)

It can be seen that the recommendation accuracy of relevant news is relatively high.

### 4.2 Personalized recommendation test

The following are 6 news items followed by the newly registered account. You can see the following topics: South Korea, cities, Chinese citizens, party elections, etc.
![enter image description here](https://i.imgur.com/dWKJDyz.png)

![enter image description here](https://i.imgur.com/EPDWHIX.png)

Here are the news recommendations for this user:
![enter image description here](https://i.imgur.com/zHuCCii.jpg)

You can see that the suggested news topics also include keywords like South Korea, cities, Chinese citizens, and so on (though a sports story was eventually mixed in).

## 5 Project review

In the implementation process of the whole project, I think the most interesting work is the news keyword clustering. Through Google's open source project Word2Vec and other frameworks, wikipedia articles can be extracted from a collection of Chinese words, training the model can actually find a word synonym, can determine whether two words are related. However, there are many problems in training the model. Since the entire project is running on rented cloud servers, the training model naturally wants to run on servers as well. However, the first few times during the training, the Python process automatically terminated, unable to complete the script. After a lot of debugging, the memory usage of the script increased sharply after it started running, because the whole text file needed to be read into the memory for processing. However, the file was too large and the server was out of memory, so the operating system automatically interrupted the Python process. Upgrading server memory costs more, so I had to run scripts on my own computer to train the model of Word2Vec before uploading it to the server. Although I am not familiar with the algorithms behind these processes, I am also interested in hot fields such as natural language processing and deep learning. If I have time in the future, I hope to learn these knowledge further and more systematically.

In addition, the suggested API provided by Microsoft cognitive services is interesting. After modeling the data set provided in the official documentation, you can see that a user's browsing history is indeed reflected in the recommended items. When applied to the project, due to lack of user access to records, so you need to write the corresponding script generates a user some records uploaded to the API server, but because of these records is randomly generated, so the feature is not particularly evident, so the final is main or use according to the user is browsing news, use the aforementioned algorithm find the news related news recommended to the user.

Through the project practice, we not only for web crawl, no database, web caching, response type page, the commonly used technology such as information push had certain understanding, at the same time also have the opportunity to can in natural language processing, recommendation system, and other areas of the cutting-edge research made some entry-level practice, so I think this project should be a very meaningful course assignments.

## 6 Reference material

[Node.js Express API document](https://expressjs.com/api.html)

["Stutter" Chinese participle: do the best Python Chinese participle phrase](https://github.com/fxsjy/jieba)

[Word2vec experiment on the corpus of Chinese and English wikipedia](http://www.52nlp.cn/%E4%B8%AD%E8%8B%B1%E6%96%87%E7%BB%B4%E5%9F%BA%E7%99%BE%E7%A7%91%E8%AF%AD%E6%96%99%E4%B8%8A%E7%9A%84word2vec%E5%AE%9E%E9%AA%8C)

[Microsoft cognitive services - suggested API documentation](https://azure.microsoft.com/zh-cn/services/cognitive-services/recommendations/)

[The official SDK documentation for the third-party push service OneSignal](https://documentation.onesignal.com/docs)

