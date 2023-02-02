# 6. Chunk Oriented Processing

One of the great strengths of Spring Batch is chunk-oriented processing.
This time, let's take a look at what chunk-oriented processing is.

## 6-1. Chunk?

Chunk in Spring Batch refers to the **number of rows processed between each commit** when working with chunks of data.
In other words, chunk-oriented processing means **reading data one at a time to create chunks called chunks, and then handling transactions in units of chunks**.

Transactions are important here.
Since transactions are performed in units of chunks, **in case of failure, only the corresponding chunk is rolled back**, and the previously committed transaction range is reflected.

Since chunk-oriented processing ultimately means that data is processed in units of chunks, the figure is as follows.

![chunk-process](./images/6/chunk-process.png)

> [Picture from the official documentation](https://docs.spring.io/spring-batch/4.0.x/reference/html/index-single.html#chunkOrientedProcessing) covers **only the individual item being processed** there is.
> Please note that the above figure is slightly different as it covers even the chunk unit.

- Read one data from Reader
- The read data is processed in the Processor
- After collecting the processed data in a separate space, when it is accumulated as much as the chunk unit, it is delivered to the Writer and the Writer saves it in batches.

**It is only necessary to remember that the Reader and Processor handle one case at a time, and the Writer handles them in chunk units**.

If chunk-oriented processing is expressed in Java code, it will look like the following.

```java

for(int i=0; i<totalSize; i+=chunkSize){ // Process in chunkSize units
     List items = new Arraylist();
     for(int j = 0; j < chunkSize; j++){
         Object item = itemReader.read()
         Object processedItem = itemProcessor.process(item);
         items. add(processedItem);
     }
     itemWriter. write(items);
}
```

**Do you understand the meaning of being grouped and processed by chunkSize**?
Now, let's find out how Chunk-oriented processing is going by looking at the actual Spring Batch internal code.

## 6-2. Peek at the ChunkOrientedTasklet

The `ChunkOrientedTasklet` class handles the entire logic of chunk-oriented processing.
Just looking at the class name, you can tell what it does at once.

![tasklet1](./images/6/tasklet1.png)

The code you need to look closely at here is `execute()`.
You can see that the entire code for working in chunk units is here.
The internal code is below.

![tasklet2](./images/6/tasklet2.png)

- Get data as much as chunk size from Reader with `chunkProvider.provide()`.
- Process (Processor) and save (Writer) data received from `chunkProcessor.process()` to Reader.

If you go to `chunkProvider.provide()` to get data, you can see how to get data.

![tasklet3](./images/6/tasklet3.png)

Call `read()` until `inputs` are accumulated as much as ChunkSize.
This `read()` actually calls `ItemReader.read` if you look inside.

![tasklet4](./images/6/tasklet4.png)

![tasklet5](./images/6/tasklet5.png)

In other words, `provide()` is to retrieve data one by one from `ItemReader.read` and accumulate data as much as the chunk size.

Now, let's take a look at how to process and store the accumulated data.

## 6-3. Peek at SimpleChunkProcessor

The `ChunkProcessor` is responsible for containing the Processor and Writer logic.

![process0](./images/6/process1.png)

Since it is an interface, it must have an actual implementation.
The one used by default is `SimpleChunkProcessor`.

![process1](./images/6/process2.png)

If you look at the class above, you can see in detail how Spring Batch handles chunk units.
The core logic responsible for processing is `process()`.
The code for this `process()` is as follows.

![process2](./images/6/process3.png)

- Receives `Chunk<I> inputs` as a parameter.
  - This data is items accumulated as much as the ChunkSize received from `chunkProvider.provide()` earlier.
- `transform()` passes the received `inputs` to ``doProcess()` and receives the converted value.
- A large amount of data processed through `transform()` is collectively saved through `write()`.
  - `write()` can be saved or sent to an external API.
  - This depends on how the developer implemented the ItemWriter.

Here, `transform()` calls `doProcess()` through a loop.
This method uses ItemProcessor's `process()`.

![process3](./images/6/process4.png)

`doProcess()` is processed. If ItemProcessor does not exist, item is returned as it is, processed with ItemProcessor's `process()` and returned.

![process4](./images/6/process5.png)

And these processed data are batch processed by calling SimpleChunkProcessor's `doWrite()` as shown above.

![process4](./images/6/process6.png)

Now, we looked at the actual code for Chunk-oriented processing and found out how it is processed.
Below, we will clear up some misunderstandings about ChunkSize.

## 6-4. Page Size vs Chunk Size

Those who have used Spring Batch in the past have probably used PagingItemReader a lot.
Among those who use PagingItemReader, some people misunderstand Page Size and Chunk Size as the same meaning.
**Page Size and Chunk Size have different meanings**.

**Chunk Size refers to the transaction unit to be processed at one time**, and **Page Size refers to the amount of items to be retrieved at one time**.

Now, let's directly look into the actual Spring Batch ItemReader code to see how the two differ.

Let's first look at the `read()` method of `AbstractItemCountingItemStreamItemReader`, which is the parent class of PagingItemReader.

![read1](./images/6/read1.png)

As you can see, if there is data to be read, `doRead()` is called.

The code for `doRead()` is as follows.

![read2](./images/6/read2.png)

In `doRead()`, `doReadPage()` is called if there is no data to be read currently or if the page size is exceeded.
When there is no data to be read, it refers to when read starts for the first time.
If the page size is exceeded, for example, if the page size is 10, the data to be read this time is the 11th data.
In this case, you can see that `doReadPage()` is called because the page size is exceeded.

In other words, it is searched by breaking **page by page**.

> If you think of paging lookup in bulletin board creation, it will be easier to understand.

Starting from `doReadPage()`, sub-implementation classes create paging queries in their own way.
Here, we will take a look at the commonly used **JpaPagingItemReader** code.

The code of `doReadPage()` of JpaPagingItemReader is as follows.

![read3](./images/6/read3.png)

Create a paging query (`createQuery()`) by specifying `offset` and `limit` values as much as the Page Size specified in Reader, and use (``query.getResultList ()`).
Query execution results are stored in `results`.
From `results` saved like this
\*\*Each time `read()` is called, it pulls out one by one and passes it on.

In other words, Page Size is a **value for specifying the size of a page in a paging query**.

What if the two values are different?
If PageSize is 10 and ChunkSize is 50, **If ItemReader sees Page 5 times, 1 transaction occurs and chunks are processed**.

Performance issues may occur because 5 query lookups occur for one transaction processing. So, in Spring Batch's PagingItemReader, the following annotation is left at the top of the class.

> Setting a fairly large page size and using a commit interval that matches the page size should provide better performance.
> (Performance will improve if you set a fairly large page size and use a commit interval that matches the page size.)

In addition to performance issues, if you use JPA, if you set the two values differently, the persistence context will be broken.
(Please refer to [resolved issues](http://jojoldu.tistory.com/146) related to the previous one)

Although the two values have different meanings, it is recommended that you match the two values because **matching the two values is a generally good method** due to the various issues mentioned above.

## 6-5. Wrap-up

This time, we took a deep look at the code of Spring Batch.
There are a lot of theoretical contents, so it might be a little difficult.
It is an important concept to understand the overall picture of Spring Batch, so we recommend that you familiarize yourself with it.
From next time, we will proceed with the example code of Spring Batch again.
Thank you for reading this long post :)
