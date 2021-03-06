# Python 'Stock Ticker' Demo Application For MongoDB Change Streams

A Python application demonstrating MongoDB's [Change Streams](https://docs.mongodb.com/manual/changeStreams/) capability, by simulating a simple "Stock Prices" system. Changes to stock prices are listened for and displayed to the user as and when these price changes are persisted in the MongoDB database. 

The MongoDB database/collection created and used by the Python application is: 'market.stocksymbols'.

## Prerequisites

1. A MongoDB deployment is already configured, running and accessible. The version of MongoDB must be 3.6 or greater. The deployment can be either a MongoDB Replica Set (see [example](https://github.com/pkdone/mongo-quick-repset) of how to quickly run one on the same local machine) or a Sharded Cluster (see [example](https://github.com/pkdone/mongo-multi-svr-generator) of how to quickly run one on the same local machine). **Note:** A standalone non-replicated MongoDB deployment cannot be used, as this is not supported by the Change Streams capability.

2. The MongoDB Python Driver (PyMongo) is already installed locally (must be PyMongo version 3.6 or greater). Example:

    ```
    $ sudo pip install pymongo
    ```

## How To Run

1. In the Python script file **mongo_stock_ticker.py**, near the top of the file, change the value of the variable **MONGODB_URL** to reflect the address of the MongoDB Replica Set or Sharded Cluster, that was established as part of the prerequisites.


    ```
    MONGODB_URL = 'mongodb://...'
    ```

2. Using a command line shell, clean out any old copy of the stock prices database data that may exist, by running the Python script with the **CLEAN** command (this is not necessary for first time running the demo, but it doesn't do any harm if run). Example:


![CLEAN](imgs/clean.png)


3. Initialise the stock prices database by running the Python script with the **INIT** command. Initialises the database collection with a set of random stock symbol and price records, plus a handful of familiar stock symbols (eg. MDB, ORCL, GOOGL) with prices. If the target environment is a Sharded cluster, automatically enables sharding on the stock prices database collection. Inserts about 20 thousand records which takes around 10 seconds for a replica-set and around 30 seconds for a sharded cluster (due to the need to perform some extra configuration operations during initialisation by the script - see FAQ at base of this README for further information). Example:


![INIT](imgs/init.png)


4. Start continuously changing records in the stock prices database collection by running the Python script with the **CHANGE** command (abort using 'Ctrl-C'). Performs random updates on any stock in the collection and also performs some random deletes and inserts. Executes approximately 16 operations per second, of which only 4 operations relate to updating the prices of the familiar stock symbols (eg. MDB, ORCL, GOOGL). Example:


![UPDATES](imgs/change.png)


5. In a separate command line shell, from where the executed command is still running from point 3, start continuously listening to change events on the stock prices database collection and printing the changes. Run the Python script with the **TRACE** command to invoke this (abort using 'Ctrl-C'). This filters the database changes to listen for updates to the prices of the familiar stock symbols records only (eg. MDB, ORCL, GOOGL), and prints out each new value as and when the change occurs. Example:


![LISTEN](imgs/trace.png)


6. In a separate command line shell, from where the executed command is still running from point 3, start continuously listening to change events on the stock prices database collection and showing each price change inline in the console, next to its respective stock symbol. Run the Python script with the **DISPLAY** command to invoke this (abort using 'Ctrl-C'). This filters the database changes to listen for updates to the prices of the familiar stock symbols records only (eg. MDB, ORCL, GOOGL), and displays each changed value, inline, as and when the change occurs. Recently changed stock prices are highlighted for a couple of seconds. Example:


![LISTEN](imgs/display.png)


## FAQ

* __Q1__. In this example application, why isn't the resume token saved somewhere by the client application listening process, so that if the client process crashes, the client could restart from where it left off?

* __A1__. The client implementation is safe because it "initial syncs" every time it starts up (after getting a resume token and before watching the collection). The logic here is that these sorts of clients (not just stock tickers specifically) may only ever be interested in a small subset of data (e.g. only a few thousand records from maybe a set of billions of total records). The trade off being that the client application may not be turned on during the night, for example, and when turned on each morning, it could either try to reproduce all the changes since the night before (it could well have fallen off oplog) or it could just "initial sync" the few 10s/100s/1000s records of interest, each time, before listening. In summary, if the client application is not another type of "datastore" (that is usually up most of the time), and the client is only ever interested in a small subset of records, it probably doesn't makes sense to persist the resume token and always try to catch-up. Instead, the client can just resync every time it re-starts.


* __Q2__. I get a 'resume token does not exist' error when listening for changes using the script's 'DISPLAY' command, when pointing at a __sharded__ cluster, why?

* __A2__. In MongoDB version 3.6 there is a Change Streams resumability issue if a sharded collection does not currently have records existing on all shards (i.e. when not every shard is the owner of at least 1 chunk from the collection - this can be seen when you run the sh.status() administrative command). As a result, when the script#s 'INIT' command is used to initialise a database that happens to be sharded, the script 'pre-splits' the chunk ranges for the empty sharded collection, to ensure at least some of the subsequent records in the collection will be stored in each of the shards, thus avoiding the issue. 


* __Q3__. When I run the script's 'TRACE' or 'DISPLAY' command to listen for changes, if pointing at a __sharded__ cluster, I see gaps of around 10 seconds with no changes shown, and then a batch of changes is shown all at once, why? 

* __A3__. In a sharded cluster, in order to guarantee total ordering across shards, the Mongos routers cannot return events until every shard's clock has progressed. If updates are continuously coming into every shard, the clock progresses organically and immediately. However, if one shard is 'cold' and not receiving any updates, a 'no-op' token is sent to the shard every 10 seconds to progress time. Therefore, if a 10 second 'stagger' is experienced when the script is run to listen for changes, this will be due to at least one shard not receiving an update for any of the subset of records it owns. To avoid this situation, the script's 'INIT' command 'pre-splits' the chunk ranges for the shards. As a result, when running the script's 'CHANGE' command, every shard will receive at least some of the updates, so the Change Streams events will be shown almost instantaneously, with no gaps/staggers.

