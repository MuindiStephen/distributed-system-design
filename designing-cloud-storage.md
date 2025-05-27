# Designing a Cloud Storage Service like Dropbox or Google Drive
Cloud file storage enables users to store their data on remote servers.

Why cloud storage?
1. **Availability:** Users can access their files from any devices, anywhere, anytime.
2. **Durability and Reliability:** Cloud storage ensures that users never lose their data by storing their data on different geographically located servers.
3. **Scalability:** Users will never have to worry about running out of storage space, as long as they are ready to pay for it.



## 1. Requirements and System Goals
What do we want to achieve from a cloud storage system?
1. Users should be able to upload/download files from any devices
2. Users should be able to share files and folders with other users
3. The service should support large files
4. The service should allow syncing of files between devices. Updating a file on a device should get synchronized on all other devices.
5. ACID-ity on all file operations should be enforced.
6. The service should support offline editing, and as soon as users come online, all changes synced.
7. The service should support snapshotting of data, so that users can go back to previous versions of it

#### Design Considerations
- We should expect large read and write volumes, with the read/write ratio being about the same.
- Files must be stored in small chunks. This has a lot of benefits, such as if a user fails to upload a file, then only the failing chunk will be retried instead of the entire file.
- We can reduce on data exchange by transferring updated chunks only. And, for small changes, clients can intelligently upload the diffs instead of the whole chunk
- The system can remove duplicate chunks, to save storage space and bandwidth.
- We can prevent lots of round trips to the server if clients keep a local copy of metadata (file name, size etc)

## 2. Capacity Estimation
```python
* Assume: 100 million users, 20 million DAU (daily active users)
* Assume: Each user has on average two devices
* Assume: Each user has on average about 100 files/photos/videos, we have 

Total files => 100,000,000 users * 100 files = 10 billion files

* Assume: The average file size => 100KB, Total storage would be: 
        0.1MB * 10B files => 1 PB(Petabyte)
```



## 3. High Level Design

The user will have a folder as their workplace on their device. Any file/photo/folder placed inside this folder will be uploaded to the cloud. If changes are made to it, it will be reflected in the same way in the cloud storage.
- We need to store files and metadata (file name, size, dir, etc) and who the file is shared with.
- We need block servers to help the clients upload/download files to our service
- We need metadata servers to facilitate updating file and user metadata.
- We need a synchronization server to notify clients whenever an update happens so they can start synchronizing the updated files.
- We need to keep metadata of files updated in a NoSQL database.

![](images/designing_cloud_high_level.png)

## 4. Component Design
The major components of the system are as follows:

### a. Client
The client monitors the workspace folder on the user's device and syncs changes to the remote cloud storage. 
The main operations for the client are:
1. Upload and download files
2. Detect file changes in workspace folder
3. Resolve conflict due to offline or concurrent updates.

#### Handling efficient file transfer
We can break files into small chunks, say 4MB. We can transfer only modified chunks instead of entire files. To get an optimal chunk size we can the following:
- Input/output operations per sec (IOPS) for our storage devices in our backend.
- Network bandwidth.
- Average file size storage.

We should also keep a record of each file and the chunks that make up that file in our metadata servers.

A copy of metadata can also be kept with the client to enable offline updates and save round trips to update the remote metadata.

#### Syncing with other clients
We can use HTTP long polling to request info from the server. If the server has no new data for this client, instead of sending an empty response, it holds the request open and waits for response information to become available. Once new info is available, the server immediately sends a HTTP response to the client, completing the open request. 

#### Major parts of the Client
![](images/designing_cloud_client.png)

1. **Internal Metadata DB:** to keep track of all files, chunks, versions, and locations in the file system.
2. **Chunker:** will split files into chunks, and reconstruct a file from its chunks.
3. **Watcher:** will monitor workspace folder and notify the indexer of user action (e.g CRUD operations), as well as listen for incoming sync changes broadcasted by `Sync Service`. 
4. **Indexer:** will process events from the watcher and update the client DB with necessary chunk/update information on files. Once chunks are synced to the cloud, the indexer can communicated with `remote Sync Service` to broadcast changes to other clients and update the `Remote Metadata DB`.

On client communication frequency:
> The client should exponentially back-off connection retries to a busy/slow server, and mobile clients should sync on demand to save on user's bandwidths/bundles and space.


### b. Metadata DB
The Metadata database can be a relational database like MySQL or a NoSQL DB like DynamoDB. 
The Sync Service should be able to provide a consistent view of the files through a DB, especially if the file is being edited by more than one user. 

If we go with NoSQL for its scalability and performance, we can support ACID properties programmatically in the logic of our Sync Service.

The objects to be saved in the Metadata NoSQL DB are as follows:
- Chunks
- Files
- User
- Devices
- Workspace Sync Folders


### c. Sync Service
This component will process file updates made by a client and apply changes to other subscribed clients. 
It will sync local DB for the client with the info store in the remote Metadata DB.

**Consistency and reliability:** When the Sync Service receives an update request, has a verification process. This process first checks with the Metadata DB for consistency before and proceeding with the update, ensuring data integrity. This step helps prevent conflicts and inconsistencies that could come about from concurrent updates from multiple clients.

**Efficient Data Transfer:** By transmitting only the diffs between file versions instead of the entire file, bandwidth consumption and cloud data storage usage are minimized. This approach is benefitial especially for large files and frequent update scenarios. 

**Optimized storage:** The server and clients can calculate a hash using a collision resistant alogorithm (SHAs, Checksums or even Merkle trees) to see whether to update a copy of a chunk or not. On the server, if we already have a chunk with a similar hash, we don't need to create another copy, we can use the same chunk. The sync service will intelligently identify and reuse existing chunks, reducing redundancy and conversing storage space

**Scalability through messaging middleware:** Adding a messaging middleware between clients and the Sync Service will allow us to provide scalable message queuing and change notifications, supporting a high number of clients using pull or push strategies.
Multiple Sync Service instances can receive requests from a global request queue, and the messaging middleware will be able to balance its load.

**Future-Proofing for Growth:** By designing the system with scalability and efficiency in mind, it can accommodate increasing demands as the user base grows or usage patterns evolve. This proactive approach minimizes the need for major architectural changes or performance optimizations down the line, leading to a more sustainable and adaptable system architecture.

### d. Message Queuing Service
This component supports asynchronous communication between client and Sync Service, and efficiently store any number of messages in a highly available, reliable and scalable queue.

The service will have two queues: 
1. **A Request Queue:** is a global queue which will receive client's request to update the Metadata DB.
From there, the Sync Service will take the message to update metadata.

2. **A Response Queue:** will correspond to an individual subscribed client, and will deliver update messages to that client. Each message will be deleted from the queue once received by a client. Therefore, we need to create separate Response Queues for each subscribed client

![](images/designing_cloud_mqs.png)

## 5. File Processing Workflow
When Client A updates a file that's shared with Client B and C, they should receive updates too. If other clients
are not online at the time of update, the Message Queue Service keeps the update notifications in response queues until they come online.

1. Client A uploads chunks to cloud storage.
2. Client A updates metadata and commits changes.
3. Client A gets confirmation and notifications are sent to Client B and C about the changes.
4. Client B and C receive metadata changes and download updated chunks.

## 6. Data Deduplication
We'll use this technique to remove duplicateed copies of data to cut down storage costs.
For each incoming chunk, we calculate a hash of it and compare it with hashes of the existing chunks to see if we have a similar chunk that's already saved.

Two ways to do this:

a. **In-line deduplication:** do hash calculations in real-time as clients enter the data on the device. If an existing chunk has the same hash as a new chunk, we store a reference to the existing chunk as metadata. This prevents us from making a full copy of the chunk, saving on network bandwidth and storage usage.

b. **Post-process deduplication:** store new chunks and later some process analyzes the data looking for duplicated chunks. The benefit here is that clients don't need to wait for hash lookups to complete storing data. This ensures there's no degradation in storage performance. The drawback is that duplicate data will consume bandwidth, and we will also unnecessarily store it, but only for a short time. 

## 7. Partitioning Metadata DB
To scale our metadata DB, we can partition it using various partition schemes:

We can use Range-based partitioning where we store files/chunks in separate partitions based on the first letter of the file path. But, later this might lead to unbalanced servers, where partitions that start with frequently occuring letters will have more files than those that dont.

For Hash-based partitioning, we can take a hash of the object and use it to determine the DB partition to save the object. A hash on the `FileID` of the File object we are storing can be used to determine the partition to store the object. 

The hashing function will distribute objects into different partitions by mapping them to a number between `[1,...,256]` and this number would be the partition we store the object. And to prevent overloading some partitions,  we can use `Constitent Hashing`.

## 8. Load Balancer
We can add the load balancer at two places:
1. Between Client and Block servers
2. Between Client and Metadata servers

![](images/designing_cloud_detailed.png)

We can have a round robin load balancer that distributes incoming requests equally among backend servers. But if a server is overloaded or slow, the LB will not stop sending new requests to that server. To handle this, a more intelligent LB strategy can be implemented such that it queries for a backend server load before it sends traffic to that server, and adjusts traffic to a server based on its current server load. 

## 9. Caching
To deal with hot frequently used files/chunks, we can create a cache for block storage. We'll store whole chunks
and the system can cechk if the cache has the desired chunk before hitting Block storage.

LRU eviction policy can be used to discard the least recently used chunk first.
We can also introduce a cache for metadata DB for hot metadata records.
