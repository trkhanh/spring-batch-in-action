#spring-batch-in-action

Spring Batch In Action has not been revised since 2011, and there is no Korean translation, and most Spring Batch articles in Korea are tutorials or official document translations, so I started it because I was frustrated.
As commerce or other service systems grow, batch work increases, and Spring Batch is often used in a rule-of-thumb way.
Spring MVC has a lot of data, but there are too many batches, so I started to organize it.

## index

Expected

- Simple Batch Job
  - Review of job execution results
  - Job, JobInstance, JobExecution, etc. BATCH_JOB schema
    - Review the contents of the BatchExecution table
  - Introduction to BatchStatus
- Batch job flow
  \*Step
  - Flow
  - split & mutli threads
  - decide
    - Step branch processing through JobExecutionDecider
- Introduction to batch steps
  - chunk oriented
  - chunk-oriented as code
  - Brief introduction of reader processor writer
- JobParameter
  - `@JobScope`, `@StepScope`, `@Value`
  - Build => Execute batch.jar => Check the parameters displayed on the console
  - Check for errors due to duplication of Job Parameters
  - Confirm that it is performed without build with test code
  - [StepScope](https://docs.spring.io/spring-batch/3.0.x/reference/html/configureStep.html)
  - [JobScope](https://docs.spring.io/spring-batch/3.0.x/reference/html/configureStep.html)
- Reader
  - Notes on overriding the `read()` method when creating a custom reader
    - If `this.data.hasNext()` is false, always return null.
    - If null is not returned, infinite reading begins
  - DB & JPA
    \*JpaItemReader
    - JpaPagingItemReader
    - IbatisReader was removed with Spring Batch 4.0
    - What if you need to query & modify the same table? ([Note](https://stackoverflow.com/questions/26509971/spring-batch-jpapagingitemreader-why-some-rows-are-not-read))
  - File
    - FlatFileItemReader
  - Never use custom repository with JpaRepository
  - Read Multiple DataSources
  - Item Stream Interface
    \*ItemStream?
    - Differences with paging
    - JpaCursorItemReader
  - QuerydslPagingItemReader & QuerydslCursorItemReader
- Writer
  - Introduction to Simple Writer
    - chunk difference
    - Introduction of precautions when chunk and page size are different
  - DB & JPA
    - JDBC writers
    - JPA Writer
  - Write Multiple DataSources
- Processor
  _ Simple processor
  _ Filter processing in Processor
  _ Validate in Processor
  _ `Validator`
  * Skip on failure with `Validator.setFilter(true)`
  *Composite
  _ When you want to combine multiple processors
  _ `CompositeItemProcessor`
  _ `List<ItemProcessor<Pay,Pay>> delegates`
  _ Processor is a concept similar to Stream in Java 8.
  _Transactions
  _ [Basics](https://blog.codecentric.de/en/2012/03/transactions-in-spring-batch-part-1-the-basics/)
  _ [Restart, Cursor-based Readers, Listeners](https://blog.codecentric.de/en/2012/03/transactions-in-spring-batch-part-2-restart-cursor-based-reading-and-listeners /)
  _ [Skip And Retry](https://blog.codecentric.de/en/2012/03/transactions-in-spring-batch-part-3-skip-and-retry/)
  _Error
  _ Restart
  * `restartable=false`?
  *Retry
  _ Skip
  _ Listeners
  \*runid
- test code
- Multi DataSource
  - Multi Datasource
  - Multi-EntityManager
  - Multi TxManager
- How about batch scheduling? (Jenkins)
