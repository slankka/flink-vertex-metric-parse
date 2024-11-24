# flink-vertex-metric-parse
A solution to parse flink vertices metrics from archived logs, to help implement a simplest log browsing service without historyserver.

🚀**Advantages**
* **Independent**: Flink History Server independent, can be executed offline or long running streaming style.
* **Flink Way to process Flink data**: Powered by Flink SQL, to produce many useful information for flink users.
* **Easy to deploy**: The information is enough already, computing progress does not require any database record support except Yarn REST service.
* **HistoryServer Mate**:  Provide log links data to log browsing lambda service serving to Flink History Server UI.

## Log browsing service
Says Flink doc, in `Log integration` section:
> Flink does not provide built-in methods for archiving logs of completed jobs
>
> 
>`historyserver.log.jobmanager.url-pattern: http://my.log-browsing.url/<jobid>`
> 
>`historyserver.log.taskmanager.url-pattern: http://my.log-browsing.url/<jobid>/<tmid>`

If we need a log browing service, jobmanager log is simple to fetch, just request to Yarn REST API. However it's a little bit difficult for taskmanager logs.

The TaskManager Log link appears on the right of vertex graph of history server.

Look the document above, the question turns to be: build a (lamdba) function, inputs: jobId and tmId, output: full http address link of each vertex.

AFAIK, the most valuable thing of history server is to provide log details of taskmanager, such as Exceptions.

## Concepts
When flink job stops or fails, `HistoryServerArchivist` will save many execution states to job history archive directory.

* `jobmanager.archive.fs.dir: hdfs:///completed-jobs`

path example:
  
* `hdfs://corp.slankka-hdfs.com/application/app-logs/flink/da12f990aba5bcdff710e96c5a409123`

the data will stored as json values in different keys, such as exceptions, the structure of json values is defined by `ArchivedJson`.
* `http://hostname:port/jobs/7684be6004e4e955c2a558a9bc463f65/exceptions`.

🎈[Concept of ArchiveJson](./Concept-of-ArchivedJson.md)

## Idea
The data of taskmanagers of each vertex are stored in keys such as : 
* `/jobs/<jobid>/vertices/<vertexid>/taskmanagers`

So, the way to collect vertices metrics of taskmanagers is to parse archived json data, each file represents a flink job.
The file name will be `<jobid>`

## Steps
1. parse the archived json files
2. extract values from keys suffix-by `taskmanagers`
3. extract fields of each vertex and taskmanagers
4. generate full taskmanager log url along with vertex id and name
5. optionally write the metrics to a storage such as database.

BTW: It's not the only way to compute results we want. 
I tried before, writing a pure web service can not avoid requesting flink history server back, it's like a dependency cycle.

## Flink SQL solution:
I found a modern way to implement, that is pure flink sql to do all the things.

[Flink SQL solution](./flink-sql-solution.md)



