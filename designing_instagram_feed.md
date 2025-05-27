# Designing Instagram Feed

A feed is a constantly updating scrollable list of posts, photos, videos, and status updates from all the people and pages a user follows.


## 1. Requirements and System Goals

### Functional requirements
- Feeds may contain images, videos and text.
- Feeds are generated from the posts belonging to the pages and people the user follows.
- The service should support appending new posts as they arrive to the feed for all active users

### Non-functional requirements 
- The system should be able to generate a user's newsfeed in real-time - maximum latency seen by the end user should be about 2s.

## 2. Capacity Estimation and Constraints
Assume on average a user has 300 friends and follows 200 pages.

**Traffic Estimates:** A typical user checks their feed about 5 times per day on average. If we have 200 million daily active users, then:

```text
Per day: 200M * 5 => 1B requests

Per second:   1B / 86400 sec => 11500 reqs/sec 
```

**Storage estimates:** Assume we have on average 500 posts for each user's feed that we want to keep in memory for fast fetching. 
For simplicity, let's also assume an average photo file size posted would be about 200KB. This means we need about 200KB X 500 = 10 MB per user. 
To store all this data for all active users, we'll need:

```text
    200 M active users * 10MB  ~= 2 Petabyte of memory
```

If a server can hold 100 GB memory, we'd need about 20000 machines to keep the top 500 posts in memory.


## 3. System APIs
> By defining the system APIs, you are explicitly stating what is expected from the system

We'll have REST APIs to expose our service's functionality.

**Getting User Feed**

```python
getUserFeed(
    api_dev_key: int,  # Key of a registered user, used to throttle users based on their allocated quota.
    user_id: int,  # The Id of the user whom the system will generate the feed.
    since_id: int,  # (Optional) Return results with IDs more recent than this ID.
    count: int ,   # (Optional) Specifies number of feed items to try and retrieve.
    max_id: int,   # (Optional) Returns results with IDs younger than the specified ID.
    exclude_replies  # (Optional) Prevents replies from appearing in the results.
```

**Returns:** (JSON) object containing a list of feed items. 

## 4. Database Design
There are three major objects: User, Entity (Business Accounts, Brands, Pages etc) and Post (A feed item). 
    * A user follows entities and other users.
    * Users and entities can both post a Post which can contain text, images, or videos
    * Each Post has a UserID of the user who created it. 
    * For simplicity, let's assume only users can create a post.
    * A Post can optionally have an EntityID that points to the page or business entity where the post was created.

If we use a relational DB, we can model two relations: User-to-Entity and Post-to-Media relation. Since each user can be friends with many people and follow a lot of entities, we can store this relation in a separate table. 

| Users |                      |
|:----:|:-----------------------|
|PK   |  **UserID: int**         |
|     | Name: varchar(32)       |
|     | Email: varchar(32)      |
|     | DOB: datetime           |
|     | CreatedAt: datetime  |
|     | LastLogin: datetime     |

&nbsp;

 | Entity |                      |
|:----:|:-----------------------|
|PK   |  **EntityID: int**         |
|     | Name: varchar(32)       |
|     | Type: int               |
|     | Email: varchar(32)           |
|     | Description: varchar(512)      |
|     | Phone: varchar(12)           |
|     | CreatedAt: datetime  |
|     | LastLogin: datetime     |


&nbsp;

| UserFollow |                      |
|:----:|:-----------------------|
|PK   |  (**UserID, EntityOrFriendID: int**)          |
|     |  Type: enum (0, 1) |

The **Type** column above identifies if an entity being followed is a user or an entity.
We can also have a table for the Media to Post relation.

&nbsp;

| Post |                      |
|:----:|:-----------------------|
|PK   |  **PostID: int**         |
|     | UserID: int |
|     | Contents: varchar(256)      |
|     | EntityID: int           |
|     | Latitute: int           |
|     | Longitude: int  |
|     | CreatedAt: datetime     |
|    |  Likes: int         |

&nbsp;

| Media |                      |
|:----:|:-----------------------|
|     | MediaID: int |
|     | Type: enum |
|     | Description: varchar(256)      |
|     | Path: int           |
|     | Latitute: int           |
|     | Longitude: int  |
|     | CreatedAt: datetime     |

&nbsp;

| PostMedia |                      |
|:----:|:-----------------------|
|PK   |  (**PostID, MediaID: int**)          |



## 5. High Level System Design
At a high level we have two system parts:
Feed generation and Feed publishing.

#### Feed Generation
Whenever our system receives a request to generate a feed for a user, we'll perform these steps:
    1. Get all UserIDs and EntityIDs that the user follows
    2. Retrieve latest and most popular posts for those IDs
    3. Rank them based on relevance to the user. This is the user's current feed.
    4. Store the feed in a cache.
    5. Return top posts to be rendered on the user's feed.
    6. On front-end, when the user reaches the end of the loaded feed, fetch the next posts from the cache server.
    7. Periodically rank and add new posts to the user's feed.
    8. Notify user that there are new posts
    
#### Feed Publishing
When the user loads her feed, she has request and pull posts from the server. When she reaches the end of her current feed, the server can push new posts.

Should the server notify the user then the user can pull, or should the server just push new posts?

At a high level, we'll have the following components:

1. Web Servers: maintain the connection to the user to allow data transfer between user client and server.
2. Application Server: executes the work of storing new posts in the DB servers, as well as retrieval from the DB and pushing the feed to the user.
3. Metadata DB and Cache: store metadata about Users, Pages, Businesses, etc
4. Post DB and Cache: to store metadata about posts and their contents
5. Video/Photo storage and Cache: Blob storage to store all media in the posts
6. Feed generation service: to get and rank relevant posts for a user and generate the feed, and store in the cache.
7. Feed notification service: to notify user that there are newer feed posts

## 6. Detailed Component Design

Let's look at generating the feed. The query would look something like this:

~~~mysql
SELECT PostID FROM Post WHERE UserID IN
    (SELECT EntityOrFriendID FROM UserFollow WHERE UserID = <user_id> AND type = 0) -- user
    UNION
SELECT PostID from Post WHERE EntityID IN
    (SELECT EntityID FROM UserFollow WHERE UserID = <user_id> AND type = 1) -- entity
ORDER BY CreatedAt DESC 
LIMIT 100
~~~
We want to avoid a direct query to the DB due to high latency. 

We also want to avoid generating the feed when a user loads the page because it will be slow and have a high latency.

Also, the server notifying about new posts to user with lots of followers could lead to heavy loads. To improve on all this, we can pre-generate the feed and store it in memory.

### Offline generation 
We can have severs dedicated to continuously generate feeds and store in memory. When a user requests for the new posts, we can simply serve it from the stored location. Therefore a user's feed is compiled not on load but on a regular basis and returned to users whenever they request it.


When the servers need to generate feed for a user, we first query to see last time the feed was generated. New feed will be generated from that time onwards.

We can store this data in a hash table where key = UserID and value is:
```c
Struct { 
    LinkedHashMap <PostID, Post> posts;
    DateTime lastGenerated;
}
```

We can store the PostIDs in a Linked HashMap (A hash table + doubly-linked list implementation), which will allow for jumping to any post at constant time but also iterate through the map easily. (The linked list maintains the order in which keys were inserted into the map)

When fetching new posts, the client sends the last PostID the user currently sees in their feed, then the server can jump to that PostID in our hashmap and return next batch of posts from there.

#### How many feeds should we store in memory?
Initially we can store 500 posts per user, but this number can be adjusted based on the usage pattern for a user. Users who never browse past 10 feeds can have 100 posts in memory.

#### Should we generate for all users?
No. Lots of users won't log in frequently.
We can use a LRU based cache to remove users from memory that haven't accessed their feed for a long time.

We can also use machine learning to pre-generate their feed based on their login patterns.

### Feed Publishing
The process of pushing a post to all followers is call **fanout**.

Two approaches to publishing:
1. **Pull model (Fanout on load):** Clients pull data either on intervals or manually when needed. The problem with this approach is
    
    a. New data might not be shown to users until they do a pull request
    
    b. Most of the time pulls return empty responses: a wasted resource that could have been avoided.
    
2. **Push model (Fanout on write):** Immediately push a post to all followers once a user posts it. Advantage here is you don't need to iterate through your friends list to get their feeds, thus significantly reducing read operations. Users have to maintain a long-poll request with the server for receiving updates. A possible problem with this approach is when a celeb user has millions of followers, the server has to push updates to a lot of people.

3. **Hybrid:** We can combine push and pull models. We stop pushing posts from celeb users with lots of followers. We can let their followers pull updates. By doing this, we can save a huge number of fanout resources. Alternatively, we can limit the push fanout to only followers who are online.

#### How many feeds should we return to client?
Say 20 per request. Also, different clients (mobile vs desktop) fetch different number of posts due to differences in screen size and bandwidth usage.

We can notify the users on desktop where data usage is cheap. For mobile devices, data usage is expensive, so we can choose not to push data but instead to let users *pull to refresh* to get new posts.

## 7. Feed Ranking
To rank posts in a newsfeed, we can use creation time of the posts. However today's
ranking algorithms are doing more to ensure important posts are ranked higher.

The idea is to select key **features** that make a post important, 
combining them and calculating the final ranking score.

These features include:
* creation time
* number of likes
* number of comments
* number of shares, 
* time of the updates

We can also check the effectiveness of the ranking system by evaluating if it has increased user retention and add revenue etc. 



## 8. Data Partitioning
#### a. Sharding Posts and metadata
Since we have a huge number of new posts every day and our read load is extremely high, we need to distribute data across multiple machines for better read/write efficiency. 

#### Sharding based on UserID
We can try storing a user's data on one server. While storing:
- Pass a UserID to our hash function that will map the user to a DB server where we'll store all of their data.
- While querying for their data, we can ask the hash function where to find it and read it from there. 

Issues:
1. What if a user is IG famous? There will be lots of queries on the server holding that user. This high load will affect the service's performance.
2. Over time, some users will have more data compared to others. Maintaining a uniform distribution of growing data across servers is quite difficult. 

#### Sharding based on PostID
A hash function maps each PostID to a random server where we can store that post.

A centralized server running the offline feed generation service will:
1. Find all the people the user follows.
2. Send a query to all DB partitions to find posts from these people.
3. Each DB server will find posts for each user, sort them by recency and return top posts to the centralized server.

The service will merge all results and sort them again to be stored in cache; ready to be retrieved whenever a user does a pull request. This solves the problem of hot users.

Issues with this approach:
- We have to query all DB partitions to find posts for a user, leading to higher latencies.

> We can improve the performance by caching hot posts in front of the DB servers.

#### Sharding based on Post creation time

Storing posts based on creation timestamp will help us to fetch all top posts quickly and we only have to query a very small set of database servers.

Issues:
- Traffic load won't be distributed. e.g when writing, all new posts will be going to one server, and remaining servers sit idle. When reading, server holding the latest data will have high load as compared to servers holding old data.

#### Sharding by PostID + Post creation time
Each PostID should be universally unique and contain a timestamp.
We can use epoch time for this. 

![](images/twitter_epoch.png)

First part of PostID will be a timestamp, second part an auto-incrementing number. We can then figure out the shard number from this PostId and store it there.

For fault tolerance and better performance, we can have two DB servers to generate auto-incrementing keys; one odd numbers and one even numbered keys.

If we assume our current epoch seconds begins now, PostIDs will look like this:



```python
epoch = 1571691220
print('epoch - autoincrement')
for i in range(1,5):
    print(f'{epoch}-{i:06}')
```

    epoch - autoincrement
    1571691220-000001
    1571691220-000002
    1571691220-000003
    1571691220-000004
