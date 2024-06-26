# Design the Twitter timeline and search

필요한 function에 따라서 상응하는 특화 모듈을 만들어 추가하면 되는것 같음. 다음의 비디오와 같이 볼것. [video](https://www.youtube.com/watch?v=Nfa-uUHuFHg&list=PL5q3E8eRUieWtYLmRU3z94-vGRcwKr9tM&index=1&ab_channel=HelloInterview-TechInterviewPreparation)

*Note: This document links directly to relevant areas found in the [system design topics](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics) to avoid duplication.  Refer to the linked content for general talking points, tradeoffs, and alternatives.*

**Design the Facebook feed** and **Design Facebook search** are similar questions.

## Step 1: Outline use cases and constraints

> Gather requirements and scope the problem.
> Ask questions to clarify use cases and constraints.
> Discuss assumptions.

Without an interviewer to address clarifying questions, we'll define some use cases and constraints.

### Use cases

#### We'll scope the problem to handle only the following use cases

Request의 주체로 크개 두가지로 나눌 수 있음.
- 사람:
  - 읽기 (User timeline, Home timeline)
  - 쓰기
  - following add/remove
  - search
- 서비스
  - notification

* **User** posts a tweet
    * **Service** pushes tweets to followers, sending push notifications and emails
* **User** views the user timeline (activity from the user)
* **User** views the home timeline (activity from people the user is following)
* **User** searches keywords
* **Service** has high availability

#### Out of scope

* **Service** pushes tweets to the Twitter Firehose and other streams
* **Service** strips out tweets based on users' visibility settings
    * Hide @reply if the user is not also following the person being replied to
    * Respect 'hide retweets' setting
* Analytics

### Constraints and assumptions

#### State assumptions

General

* Traffic is not evenly distributed
* Posting a tweet should be fast
    * Fanning out a tweet to all of your followers should be fast, unless you have millions of followers
* 100 million active users
* 500 million tweets per day or 15 billion tweets per month
    * Each tweet averages a fanout of 10 deliveries
    * 5 billion total tweets delivered on fanout per day
    * 150 billion tweets delivered on fanout per month
* 250 billion read requests per month
* 10 billion searches per month

Timeline

* Viewing the timeline should be fast
* Twitter is more read heavy than write heavy
    * Optimize for fast reads of tweets
* Ingesting tweets is write heavy

Search

* Searching should be fast
* Search is read-heavy

#### Calculate usage

**Clarify with your interviewer if you should run back-of-the-envelope usage calculations.**

* Size per tweet:
    * single char (ASCII) - 1 byte
    * Emojis - 4 bytes
    * `tweet_id` - 8 bytes
    * `user_id` - 32 bytes
    * `text` - 140 bytes
    * `media` - 10 KB average
    * Total: ~10 KB
* 150 TB of new tweet content per month
    * 10 KB per tweet * 500 million tweets per day * 30 days per month
    * 5.4 PB of new tweet content in 3 years
* 100 thousand read requests per second
    * 250 billion read requests per month * (400 requests per second / 1 billion requests per month)
* 6,000 tweets per second
    * 15 billion tweets per month * (400 requests per second / 1 billion requests per month)
* 60 thousand tweets delivered on fanout per second
    * 150 billion tweets delivered on fanout per month * (400 requests per second / 1 billion requests per month)
* 4,000 search requests per second
    * 10 billion searches per month * (400 requests per second / 1 billion requests per month)

Handy conversion guide:

* 2.5 million seconds per month
* 1 request per second = 2.5 million requests per month
* 40 requests per second = 100 million requests per month
* 400 requests per second = 1 billion requests per month

## Step 2: Create a high level design

> Outline a high level design with all important components.

![Imgur](http://i.imgur.com/48tEA2j.png)

## Step 3: Design core components

> Dive into details for each core component.





두가지 timeline 종류가 있음. 
- User timeline: 개별 유저가 올린 tweet들만 모아서 볼 수 있음. Tweeter에서 특정 유저의 아이디를 눌러서 들어가면 그 사람이 쓴 tweet들만 보이는 기능.
   - SQL을 사용. user timeline은 home timeline에 비해서 상대적으로 write 수가 적음. 유저들이 tweet을 올리면, userID, tweetID같은 정보들을 시간순으로 그냥 SQL에 저장하면 됨. 그리고 SQL은 ACID에서 consistency가 중요하게 작용함. 유저 timeline에서는 데이터를 놓치지 않고 전달하는것이 중요함. 특정 유저에 대한 user timeline이 필요할 경우, SQL에서 미리 indexing 해 놓은 userID를 이용해서 해당 유저에 대한 tweet만 빠르게 반환 할 수 있음. (SQL에 바로 userID, tweetID, contents가 같이 저장됨. media file들은 따로 noSQL에 저장). 
- Home timeline: 내가 follow하는 유저들이 올린 업데이트를 모아서 볼 수 있음.
  - home timeline은 user timeline보다 더 빠르고 많은 업데이트가 필요함 (SLQ, noSQL단위에서의 업데이트를 말하는게 아니라, timeline 레벨에서의 업데이트를 말하는것임. user timeline은 그냥 해당 유저가 포스팅을 하는 속도에만 연관되어 업데이트가 필요하지만, home timeline은 해당 유저 뿐 아니라, following 하는 모든 유저들의 속도에 영향을 받음. 그래서 더 복잡하고, scalability가 필요함, 대신 consistency는 좀 희생되어도 상관없음. 이와같은 이유로 noSQL을 사용.).
  - home timeline을 구현할 때는 개별 유저들에 대해서 그들이 following하는 유저들의 update를 모아놓은 pre-built home timeline이 존재함. 이를 위해 fan-out이 사용되는데, 이것은 새로운 post가 올라가면, 모든 following 유저들의 feed에 바로 write 되는것을 말함. user timeline에서는 이렇게 user별로 pre-built하는게 아니라, indexing을 통해서 retrival을 통해서 빠르게 정보를 얻어왔었음. 


### API design

#### 1. Post a tweet
```
POST /api/v1.0/tweets
```

Request Body
```
{
  "user_id": "123456",
  "content": "Hello, world!",
  "media_ids": ["img123", "vid456"]  // Optional: IDs of any media attached
}
```

Response
```
{
  "tweet_id": "78910",
  "status": "created",
  "created_at": "2024-05-15T12:34:56Z"
}
```

#### 2. Read User Timeline
```
GET /api/v1.0/users/{user_id}/timeline
```

Response
```
{
  "user_id": "123456",
  "tweets": [
    {
      "tweet_id": "78910",
      "content": "Hello, world!",
      "created_at": "2024-05-15T12:34:56Z"
    },
    {
      "tweet_id": "78911",
      "content": "Another post",
      "created_at": "2024-05-14T11:22:33Z"
    }
  ]
}
```

#### 3. Read Home timeline
```
GET /api/v1.0/users/{user_id}/home_timeline
```

Response
```
{
  "user_id": "123456",
  "tweets": [
    {
      "tweet_id": "90123",
      "user_id": "654321",
      "content": "Welcome to my feed!",
      "created_at": "2024-05-15T13:45:56Z"
    }
  ]
}
```

#### 4. User Follow / Unfollow

```
POST /api/v1.0/users/{user_id}/follow
DELETE /api/v1.0/users/{user_id}/follow
```

Request body
```
{
  "follower_id": "123456"  // The ID of the user who is performing the follow action.
}
```

Response
```
{
  "status": "success",
  "message": "User 123456 is now following user {user_id}."
}
```
```
{
  "status": "success",
  "message": "User 123456 has unfollowed user {user_id}."
}
```

#### 5. Search
```
GET /api/v1.0/search?query=hello
```

Response
```
{
  "query": "hello",
  "results": [
    {
      "tweet_id": "78910",
      "user_id": "123456",
      "content": "Hello, world!",
      "created_at": "2024-05-15T12:34:56Z"
    }
  ]
}
```


### DB design

#### SQL

 * Table: 'User'
   * Fields: user_id (Primary Key), username, email, password_hash, created_at, last_login, profile_description, profile_image_url
 * Table 'Follows'
   * Fields: follower_id (Foreign Key), followed_id (Foreign Key), created_at
 * Table 'Tweets'
   * Fields: tweet_id (Primary Key), user_id (Foreign Key), content, timestamp, in_reply_to_tweet_id (Nullable, Foreign Key), is_retweet (Boolean)

 #### NoSQL

 * Tweet Feed/Timeline
   * Type: Key-Value or Document Store
   * Data: { "user_id": "12345", "tweets": [{"tweet_id": "123", "timestamp": "2020-01-01T00:00:00Z"}, {...}] }
   * home timeliine을 이렇게 저장하고, user timeline은 그냥 SQL에서 해당 user_id로 찾아서 반환하는듯.
   * timeline 저장시에는 그냥 tweet_id만 저장하면 되는듯. home timeline에 저장된 tweet_id들을 저장해 두고, 나중에 request가 들어오면, user_info_service, tweet_info_service에서 필요한 정보를 받아와서 사용하는것 같음.
   * 여기서 key는 user_id, value는 tweets.
 * Reverse Search Index
   * NoSQL에 저장.
   * ElasticSearch가 이걸 위한거라는데?


### Use case: User posts a tweet

We could store the user's own tweets to populate the user timeline (activity from the user) in a [relational database](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms).  We should discuss the [use cases and tradeoffs between choosing SQL or NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql).

Delivering tweets and building the home timeline (activity from people the user is following) is trickier.  Fanning out tweets to all followers (60 thousand tweets delivered on fanout per second) will overload a traditional [relational database](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms).  We'll probably want to choose a data store with fast writes such as a **NoSQL database** or **Memory Cache**.  Reading 1 MB sequentially from memory takes about 250 microseconds, while reading from SSD takes 4x and from disk takes 80x longer.<sup><a href=https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know>1</a></sup>

We could store media such as photos or videos on an **Object Store**.

* The **Client** posts a tweet to the **Web Server**, running as a [reverse proxy](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* The **Web Server** forwards the request to the **Write API** server
* The **Write API** stores the tweet in the user's timeline on a **SQL database**
* The **Write API** contacts the **Fan Out Service**, which does the following:
    * Queries the **User Graph Service** to find the user's followers stored in the **Memory Cache**
    * Stores the tweet in the *home timeline of the user's followers* in a **Memory Cache**
        * O(n) operation:  1,000 followers = 1,000 lookups and inserts
    * Stores the tweet in the **Search Index Service** to enable fast searching
      * 간단하게 생각하면, tweet post시에 나중에 해당 tweet을 빠르게 찾기 위해서 reverse indexing을 만들거나 hashtag indexing을 수행한다고 보면 됨. 
    * Stores media in the **Object Store**
    * Uses the **Notification Service** to send out push notifications to followers:
        * Uses a **Queue** (not pictured) to asynchronously send out notifications

**Clarify with your interviewer how much code you are expected to write**.

If our **Memory Cache** is Redis, we could use a native Redis list with the following structure: (아래 코드는 home timeline을 위한 데이터가 어떻게 저장되어 있는지를 보여주는것임. 특정 유저의 home timeline에 다음과 같은 tweet_id들이 들어갈것이라는게 저장되어 있음. 

```
           tweet n+2                   tweet n+1                   tweet n
| 8 bytes   8 bytes  1 byte | 8 bytes   8 bytes  1 byte | 8 bytes   8 bytes  1 byte |
| tweet_id  user_id  meta   | tweet_id  user_id  meta   | tweet_id  user_id  meta   |
```

The new tweet would be placed in the **Memory Cache**, which populates the user's home timeline (activity from people the user is following).

We'll use a public [**REST API**](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest):

```
$ curl -X POST --data '{ "user_id": "123", "auth_token": "ABC123", \
    "status": "hello world!", "media_ids": "ABC987" }' \
    https://twitter.com/api/v1/tweet
```

Response:

```
{
    "created_at": "Wed Sep 05 00:37:15 +0000 2012",
    "status": "hello world!",
    "tweet_id": "987",
    "user_id": "123",
    ...
}
```

For internal communications, we could use [Remote Procedure Calls](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc).

### Use case: User views the home timeline

* The **Client** posts a home timeline request to the **Web Server**
* The **Web Server** forwards the request to the **Read API** server
* The **Read API** server contacts the **Timeline Service**, which does the following:
    * Gets the timeline data stored in the **Memory Cache**, containing tweet ids and user ids - O(1) (twee_id같이 간단한 데이터만 받아오는거지, text, video image같은건 나중에 받아오는것임. 여기가 아니라. )
    * Queries the **Tweet Info Service** with a [multiget](http://redis.io/commands/mget) to obtain additional info about the tweet ids - O(n)
      * Tweet에 대한 추가적인 정보 (like, retweet, reply counts, has any attached media)가 필요함. 이런 정보는 tweet id에 저장되는게 아니라 따로 저장되는듯. 
    * Queries the **User Info Service** with a multiget to obtain additional info about the user ids - O(n)
      * home timeline에 들어갈 tweet을 작성한 유저들의 추가적인 정보 (프로필, 유저 사진, 등등)이 필요함. 

REST API:

```
$ curl https://twitter.com/api/v1/home_timeline?user_id=123
```

Response:

```
{
    "user_id": "456",
    "tweet_id": "123",
    "status": "foo"
},
{
    "user_id": "789",
    "tweet_id": "456",
    "status": "bar"
},
{
    "user_id": "789",
    "tweet_id": "579",
    "status": "baz"
},
```

### Use case: User views the user timeline

* The **Client** posts a user timeline request to the **Web Server**
* The **Web Server** forwards the request to the **Read API** server
* The **Read API** retrieves the user timeline from the **SQL Database**

The REST API would be similar to the home timeline, except all tweets would come from the user as opposed to the people the user is following.

### Use case: User searches keywords

* The **Client** sends a search request to the **Web Server**
* The **Web Server** forwards the request to the **Search API** server
* The **Search API** contacts the **Search Service**, which does the following:
    * Parses/tokenizes the input query, determining what needs to be searched
        * Removes markup
        * Breaks up the text into terms
        * Fixes typos
        * Normalizes capitalization
        * Converts the query to use boolean operations
    * Queries the **Search Cluster** (ie [Lucene](https://lucene.apache.org/)) for the results:
        * [Scatter gathers](https://github.com/donnemartin/system-design-primer#under-development) each server in the cluster to determine if there are any results for the query
        * Merges, ranks, sorts, and returns the results

REST API:

```
$ curl https://twitter.com/api/v1/search?query=hello+world
```

The response would be similar to that of the home timeline, except for tweets matching the given query.

## Step 4: Scale the design

> Identify and address bottlenecks, given the constraints.

![Imgur](http://i.imgur.com/jrUBAF7.png)

**Important: Do not simply jump right into the final design from the initial design!**

State you would 1) **Benchmark/Load Test**, 2) **Profile** for bottlenecks 3) address bottlenecks while evaluating alternatives and trade-offs, and 4) repeat.  See [Design a system that scales to millions of users on AWS](../scaling_aws/README.md) as a sample on how to iteratively scale the initial design.

It's important to discuss what bottlenecks you might encounter with the initial design and how you might address each of them.  For example, what issues are addressed by adding a **Load Balancer** with multiple **Web Servers**?  **CDN**?  **Master-Slave Replicas**?  What are the alternatives and **Trade-Offs** for each?

We'll introduce some components to complete the design and to address scalability issues.  Internal load balancers are not shown to reduce clutter.

*To avoid repeating discussions*, refer to the following [system design topics](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics) for main talking points, tradeoffs, and alternatives:

* [DNS](https://github.com/donnemartin/system-design-primer#domain-name-system)
* [CDN](https://github.com/donnemartin/system-design-primer#content-delivery-network)
* [Load balancer](https://github.com/donnemartin/system-design-primer#load-balancer)
* [Horizontal scaling](https://github.com/donnemartin/system-design-primer#horizontal-scaling)
* [Web server (reverse proxy)](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* [API server (application layer)](https://github.com/donnemartin/system-design-primer#application-layer)
* [Cache](https://github.com/donnemartin/system-design-primer#cache)
* [Relational database management system (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms)
* [SQL write master-slave failover](https://github.com/donnemartin/system-design-primer#fail-over)
* [Master-slave replication](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [Consistency patterns](https://github.com/donnemartin/system-design-primer#consistency-patterns)
* [Availability patterns](https://github.com/donnemartin/system-design-primer#availability-patterns)

The **Fanout Service** is a potential bottleneck.  Twitter users with millions of followers could take several minutes to have their tweets go through the fanout process.  This could lead to race conditions with @replies to the tweet, which we could mitigate by re-ordering the tweets at serve time.

We could also avoid fanning out tweets from highly-followed users.  Instead, we could search to find tweets for highly-followed users, merge the search results with the user's home timeline results, then re-order the tweets at serve time.

Highly followed user가 post를 할 경우, 해당 유저를 follow하는 무수한 follower들에 대해서 fan-out을 수행해야 하는데, 이는 시간이 너무 오래 걸린다. 그래서, 이런 유저에 대해서는, 다른 유저들과는 다르게, post시에 follower들의 home feed에 바로 저장을 하지 말고, follower들이 home feed를 보고자 할 때, highly followed user의 tweet을 검색해서 follower의 feed에 집어넣고 time에 대해서 sorting을 수행한다. 이렇게 해서 전체 시스템에 걸리는 부하를 줄일 수 있는듯. 

Additional optimizations include:

* Keep only several hundred tweets for each home timeline in the **Memory Cache**
* Keep only active users' home timeline info in the **Memory Cache**
    * If a user was not previously active in the past 30 days, we could rebuild the timeline from the **SQL Database**
        * Query the **User Graph Service** to determine who the user is following
        * Get the tweets from the **SQL Database** and add them to the **Memory Cache**
* Store only a month of tweets in the **Tweet Info Service**
* Store only active users in the **User Info Service**
* The **Search Cluster** would likely need to keep the tweets in memory to keep latency low

We'll also want to address the bottleneck with the **SQL Database**.

Although the **Memory Cache** should reduce the load on the database, it is unlikely the **SQL Read Replicas** alone would be enough to handle the cache misses.  We'll probably need to employ additional SQL scaling patterns.

The high volume of writes would overwhelm a single **SQL Write Master-Slave**, also pointing to a need for additional scaling techniques.

* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

We should also consider moving some data to a **NoSQL Database**.

## Additional talking points

> Additional topics to dive into, depending on the problem scope and time remaining.

#### NoSQL

* [Key-value store](https://github.com/donnemartin/system-design-primer#key-value-store)
* [Document store](https://github.com/donnemartin/system-design-primer#document-store)
* [Wide column store](https://github.com/donnemartin/system-design-primer#wide-column-store)
* [Graph database](https://github.com/donnemartin/system-design-primer#graph-database)
* [SQL vs NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

### Caching

* Where to cache
    * [Client caching](https://github.com/donnemartin/system-design-primer#client-caching)
    * [CDN caching](https://github.com/donnemartin/system-design-primer#cdn-caching)
    * [Web server caching](https://github.com/donnemartin/system-design-primer#web-server-caching)
    * [Database caching](https://github.com/donnemartin/system-design-primer#database-caching)
    * [Application caching](https://github.com/donnemartin/system-design-primer#application-caching)
* What to cache
    * [Caching at the database query level](https://github.com/donnemartin/system-design-primer#caching-at-the-database-query-level)
    * [Caching at the object level](https://github.com/donnemartin/system-design-primer#caching-at-the-object-level)
* When to update the cache
    * [Cache-aside](https://github.com/donnemartin/system-design-primer#cache-aside)
    * [Write-through](https://github.com/donnemartin/system-design-primer#write-through)
    * [Write-behind (write-back)](https://github.com/donnemartin/system-design-primer#write-behind-write-back)
    * [Refresh ahead](https://github.com/donnemartin/system-design-primer#refresh-ahead)

### Asynchronism and microservices

* [Message queues](https://github.com/donnemartin/system-design-primer#message-queues)
* [Task queues](https://github.com/donnemartin/system-design-primer#task-queues)
* [Back pressure](https://github.com/donnemartin/system-design-primer#back-pressure)
* [Microservices](https://github.com/donnemartin/system-design-primer#microservices)

### Communications

* Discuss tradeoffs:
    * External communication with clients - [HTTP APIs following REST](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)
    * Internal communications - [RPC](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)
* [Service discovery](https://github.com/donnemartin/system-design-primer#service-discovery)

### Security

Refer to the [security section](https://github.com/donnemartin/system-design-primer#security).

- Rate limiter
- Encryption
- Authentication

### Latency numbers

See [Latency numbers every programmer should know](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know).

### Ongoing

* Continue benchmarking and monitoring your system to address bottlenecks as they come up
* Scaling is an iterative process
