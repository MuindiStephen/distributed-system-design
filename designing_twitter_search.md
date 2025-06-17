# Designing Twitter Search

We'll design a service that can effectively store and query user tweets.

## 1. Requirements and System Goals
- Assume Twitter has 1.5 billion total users with 800 million daily active users.
- On average Twitter gets 400 million tweets every day.
- Average size of a tweet is 300 bytes.
- Assume 500M searches a day.
- The search query will consist of multiple words combined with AND/OR.


## 2. Capacity Estimation and Constraints

```
    400 million new tweets each day,
    Each tweet is on average 300 bytes
    400M * 300 =>  120GB/day

    Total storage per second:
        120 GB / (24 hours / 3600 seconds)  ~= 1.38MB/second
```


## 3. System APIs
We can have REST APIs expose the functionality of our service.

```python

search(
    api_dev_key: string,  # The API developer key of a registered account, this will be used for things like throttling users based on their allocated quota.
    search_terms: string,  # A string containing the search terms.
    max_results_to_return: number,  # Number of tweets to return.
    sort: number,  # optional sort mode: Last first(0 - default), Best mached (1), Most liked (2)
    page_token: string,  # This token specifies a page in the result set that should be returned.
)
```
Returns: (JSON)
```
A JSON containing info about a list of tweets matching the search query.
Each result entry can have the user ID & name, tweet text, tweet ID, creation time, number of likes, etc.
```



## 4. Detailed Component Design
1. Since we have a huge amount of data, we need to have a data partitioning scheme that'll efficiently distribute the data across multiple servers.


5 year plan
```
        120 GB/day * 365 days * 5 years ~= 200TB

```

We never want to be more than 80% full at any time, so we'll need approximately 250TB storage. Assuming we also need to keep an extra copy for fault tolerance, then, our total storage will be 500 TB.

Assuming modern servers store up to 5TB of data, we'd need 100 such servers to hold all the data for the next 5 years.

Let's start with simplistic design where we store tweets in a PostgreSQL DB. Assume a table with two columns: TweetID, and TweetText.
Partitioning can be based on TweetID. If our TweetIDs are unique system wide, we can define a hash function that can map a TweetID to a storage server where we can store that tweet object.

#### How can we create system wide unique TweetIDs?
If we're getting 400M tweets per day, then in the next five years?
```
        400 M * 365 * 5 years => 730 billion tweets
```
We'll need 5 bytes number to identify TweetIDs uniquely. Assume we have a service that will generate unique TweetIDs whenever we need to store an object. We can feed the TweetID to our hash function to find the storage server and store the tweet object there.

2. **Index:** Since our tweet queries will consist of words, let's build the index that can tell us which words comes in which tweet object.


Assume:
- Index all English words,
- Add some famous nouns like People names, city names, etc
- We have 300K English words, 200K nouns, Total 500K.
- Average length of a word = 5 characters.

```
        If we keep our index in memory, we need:

        500K * 5 => 2.5 MB
```

Assume:
    - We keep the index in memory for all tweets from our last two years.
```
   Since we'll get 730 Billion tweets in the next 5 years,

   292Billion (2 year tweets) * 5 => 1460 GB
```

So our index would be like a big distributed hash table, where 'key' would be the word and 'value' will be a list of TweetIDs of all those tweets which contain that word.

Assume:
    - Average of 40 words in each tweet,
    - 15 words will need indexing in each tweet, since we won't be indexing prepositions and other small words (the, in, an, and)

> This means that each TweetID will be stored 15 times in our index.

so total memory we will need to store our index:
```
        (1460 * 15) + 2.5MB  ~=  21 TB
```
> Assuming a high-end server holds 144GB of memory, we would need 152 such servers to hold our index.

## Sharding our Data


#### Sharding based on Words:
While building the index, we'll iterate through all words of a tweet and calculate the hash of each word to find the server where it would be indexed. To find all tweets containing a specific word we have to query only server which contains this word.

Issues with this approach:
- If a word becomes hot? There will be lots of queries (high load) on the server holding the word, affecting the service performance.
- Over time, some words can end up storing a lot of TweetIDs compared to others, therefore maintaining a uniform distribution of words while tweets are growing is tricky.

To recover from this, we can repartition our data or use [Consistent Hashing](https://en.wikipedia.org/wiki/Consistent_hashing)


#### Sharding based on tweet object
While storing, we will pass the TweetID to our hash function to find the server and index all words of the tweet on that server.
While querying for a particular word, we'll query all servers, and each server will return a set of TweetIDs. A centralized server will aggregate these results to return them to the user.

![](images/sharding_based_on_tweet_object.png)

## 6. Fault Tolerance
We can have a secondary replica of each server and if the primary one dies, it can take control after the failover.
Both primary and secondary servers will have the same copy of the index.

How can we efficiently retrieve a mapping between tweets and the index server? We have to build a reverse index that will map all the tweetID to their index server. We'll keep this in the Index-Builder server.

- build a Hashtable, where key = index server number and value = HashSet containing all TweetIDs being kept at that index server.
- A HashSet will help us to add/remove tweets from our index quickly.

So whenever an index server has to rebuild itself, it can simply ask the Index-Builder server for all tweets it needs to store and then fetch those tweets to build the index. We should also have a replica of the Index-builder server for fault tolerance.

## 7. Caching
We can introduce a cache server in front of our DB. We can also use Memcached, which can store all hot tweets in memory. App servers before hitting the backend DB, can quickly check if the cache has that tweet. Based on clients' usage patterns, we can adjust how many cache servers we need. For cache eviction policy, Least Recently Used (LRU) seems suitable.

## 8. Load Balancing
Add LB layers at two places:
1. Between Clients and Application servers,
2. Between Application servers and Backend server.

LB approach:
- Use round robin approach: distrubute incoming requests equally among servers.
- Simple to implement and no overhead
- If as server is dead, LB will take it out of rotation and stop sending traffic to it
- Problem is if a server is overloaded, or slow, the LB will not stop sending new requests to it. To fix this, a more intelligent LB solution can be placed that periodically queries the server about the load and adjust traffic based on that.
