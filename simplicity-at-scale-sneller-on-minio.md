
# Introduction

While at Minio, perhaps the most important lesson that I learned was about simplicity which is paramount if you want to achieve (true) scalability. 
This permeated Minio's architecture and software design from front to back and drove, amongst others, the important decisions that I will describe next.

Firstly, a key decision was to avoid the use of any (external) storage system or database for meta data. While maybe appealing at first sight in order to "get started quickly", this adds a significant amount of complexity which becomes all the more challenging at tera- or petabyte scale and beyond. Imagine having "your" software do its job but having another dependent system becoming the bottleneck and/or the critical dependency, you will, whether you like it or not, end up becoming an expert on this system (and perhaps even more so than your own system and software).

Secondly, sticking to a "single binary" approach greatly simplifies setup and installation (and effecively pretty much _avoids_ installation altogether). By simply downloading an executable you are good to go and the same executable can run on everything from an edge device (or even a Raspberry Pi) to the largest server installation that is delivering 100 GB/sec of networking throughput. Also there are no parameters to tune and the same binary can be used in the different modes of operation that minio allows (file system mode vs erasure coding mode).

Thirdly, for Minio multi-server mode we decided to build our own distribibuted locking mechanisms. While there were various concensus protocol mechanisms out there, we evaluated all of them but in the end decided that none of these fitted our architecture and would essentially be overkill for the "simple" mechanism that we were looking for. Plus using any one of these systems would violate our single binary approach and hugely complicate the installation process and potentially become a maintenance burden.

And last but not least, by strictly adhering to a "pure" Golang approach means that we could keep our development process nimble and agile (to this day you can `git clone` minio and build it in _seconds_). But in order to do so we had to avoid any dependencies on tools like `cgo` to include high performance algorithms. Given that erasure coding and bit-rot detection are critical algorithms for any object storage system, we spend a considerable amount of effort in optimizing these algortihms in Golang native assembly for various architectures such as Intel, ARM, and PowerPC (see [here](https://blog.min.io/intel_vs_gravitron/) if you want to learn more).

## Sneller's guiding principles 

Sneller is a high performance SQL engine designed specifically for JSON. With Sneller, you can run interactive SQL on large TB-sized datasets of deeply nested semi-structured JSON stored on S3. Target use cases are event data pipelines such as Security, Observability, Ops, User Events and Sensor/IoT.

Sneller's performance derives from pervasive use of SIMD, specifically AVX-512 assembly in its 250+ core primitives. The main engine is capable of processing many lanes in parallel per core for very high processing throughputs.

![sneller language overview](https://github.com/SnellerInc/blogs/blob/main/assets/sneller-language-overview.png)

Below I would like to describe the guiding principles for Sneller's architecture:

### Thou shall not use local disk

Right from the get go, we decided that we were not going to use any form of local disk storage such as NVMe or SSD. As you will understand, based on past experience we knew exactly how fast object storage is when used in the right manner (see, for instance, this [benchmark]( https://blog.min.io/nvme_benchmark/)). Our cloud is just a pool of cores, RAM and fast networking, that's all.

### Thou shall own your data (or: we shall _not_ own your data)

Given our exclusive reliance on object storage for all data and state, we decided that all data should remain under our customer's control. That is, we are not going to import or ingest potentially petabytes of data into "our" cloud, but rather work with the data in place in our customer's buckets. The big advantage to our customers is that they remain in full control of their own data which is precisely how it should be. Of course it also means that we cannot "hold" our customer's hostage by making data export intentionally hard and expensive (all the more important the larger the scale). 

### Thou shall not need to plan

One of the best aspects of the S3 API is the _absense_ of a "free" call. Not only has S3 or object storage in general incredible durability and resiliency (very hard to achieve if you build in your own replication or so), the fact that you no longer need to worry about running "out of space" is incredibly liberating. 

In practice, while building your architecture, it is really hard to accurately predict the usage patterns and how this will impact both the generation of data (=storage) and well as the processing of data. Keeping all state on object storage truly decouples storage from compute and allows you to scale both fully independently depending on actual usage rather than some fixed or predefined ratio (which may or may not be what you need, and more often is not what you need).

### Thou shall not require a schema

The modern data stack has converged on JSON over REST APIs and JSON has become the de-facto "lingua franca" of the web. Increasingly APIs allow custom payloads that will vary from record to record and even (accidently) change "types" for identical fields or column names. This can either lead to phenomenoms such as "mapping explosions" or "schema drift" which can be very hard to detect at scale (and when you have detected it, you may very well already have lost significant amounts of data).

Fundamentally with JSON the only way to really deal with this is to be 100% schema-on-read. That is, be able to handle multiple types while querying the data. That is why Sneller supports queries in the form of `WHERE foo.bar = 1 OR foo.bar = "one" OR foo.bar = TRUE` and pick the correct comparison based on the type of the field while iterating over the data.

### Thou shall not index

Traditionally database/querying systems have relied on all sort of indexes in order to speed up queries. While these approaches do work, at scale they start to lead to a couple of issues. Firstly it will typically lead to some level of amplicification of the data that is being stored (which, granted, is not a major problem on object storage), but more importantly leads to the question of _which_ fields are you going to index? 

Especially for high-cardinality datasets (eg. the GitHub archive data has 100s of fields, most of them nested), the "all" answer becomes prohibitively expensive and would lead to excruciatingly slow ingestion rates. Therefore you are most likely going to be forced to make up-front choices as the "designer" of the table which may very well not be those fields that the users of your table might want to use or find interesting.

As such, apart from a very sparse timestamp based index (a single min and max per 100 MB of data), Sneller does not do any indexing and relies on the power and speed of its AVX-512 SIMD primitives to execute its queries. 

##  Sneller is *less

Following the popular saying "less is more", the Sneller architecture can be characterized by the following properties and benefits:

- **serverless:** stop thinking in individual servers that are running 24x7 just for you. Start thinking in always-on and pay-as-you-go models based on the amount of data that you query. Let Sneller handle the dynamic up or down scaling based on overall query demand across all tenants (and behind the scenes).

- **stateless:** by relying exclusively on object storage for any persistent data, it becomes impossible to corrupt or lose data due to any query server bugs or anomalies. In case of a query node crash, simply spin up another node and have it reload its portion of the data from S3 and continue uninterrupted.

- **schemaless:** there is no need to define up-front schema's and Sneller will seamlessly handle any schema drift. You no longer need ETL or ELT just to transform or reshape JSON to an alternative format.

- **branchless:** Sneller's performance comes from its _branchless_ SIMD code that allows many lanes to be processed in parallel on a single CPU core.

- **indexless:** by avoiding indexing, simplify and speed up the ingestion/syncing process and allow for massive scale in a cost effective manner.

## Sneller on Minio

If you want to try it out yourself, it is super easy to spin up a stack of "Sneller on Minio" which will basically give you all the tools to handle JSON at scale. Essentially using just `curl` you can submit your SQL queries to the REST API as follows:

```
$ curl -G -H "Authorization: Bearer $SNELLER_TOKEN" --data-urlencode "database=gha" \
    --data-urlencode 'json' --data-urlencode 'query=SELECT type, COUNT(*) FROM gharchive GROUP BY type ORDER BY COUNT(*) DESC' \
    'http://localhost:9180/executeQuery'
{"type": "PushEvent", "count": 1303922}
{"type": "CreateEvent", "count": 261401}
...
{"type": "GollumEvent", "count": 4035}
{"type": "MemberEvent", "count": 2644}
```

Using [docker](https://docs.sneller.io/docker.html) you can either do so on your own laptop or if you are more interested in a development or testing environment, you can try out the [Kubernetes](https://docs.sneller.io/kubernetes.html) integration.

## Open source

We have open sourced the core of the Sneller technology (including all the [AVX-512](https://github.com/SnellerInc/sneller/blob/master/vm/evalbc_amd64.s) code).

If you want to learn more, head over to the Sneller repository on GitHub at https://github.com/SnellerInc/sneller (and you are welcome to star it if you like what you see 😉.)

## Conclusion 

Object storage is phenomenally powerful to manage the largest quantities of data in the Cloud and has established itself as the de-facto standard to do so.

Combining this with a high performance serverless query engine allows for true separation of storage and compute and provides maximum flexibility for users to scale each fully independently to its own optimum.
