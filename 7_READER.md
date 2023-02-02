#7.ItemReader

Through the previous courses, we learned that Spring Batch is chunk-oriented and consists of Jobs and Steps.
Steps are processed in units of tasklets, and among tasklets, chunks are processed through **ChunkOrientedTasklet**.

> In other words, a bundle of ItemReader & ItemWriter & ItemProcessor is also a story of a Tasklet.
> This is because a bunch of them are managed by the ChunkOrientedTasklet.

From this time, we will learn these 3 elements step by step.

## 7-1. Introduction to ItemReader

Spring Batch's Chunk Tasklet proceeds through the following process.

![chunk](./images/7/chunk.png)

This time, let's look at the first part of this process, Reader.
As you can see in the picture, Spring Batch's ItemReader **reads data**.
That doesn't necessarily mean only the data in the DB.

Other data sources such as File, XML, and JSON can be used as input for batch processing.
It also supports other types of data sources such as Java Message Service (JMS).

In addition, **If you need a reader that is not supported by Spring Batch, you can create your own reader**.
Spring Batch provides an easy to create custom reader implementation for this purpose.

In summary, the data types that can be read from Spring Batch's Reader are as follows.

- read from input data
- read from file
- Read from database
- Read from other sources such as Java Message Service
- Read with your own customized Reader

Let's take a look at the implementations of this ItemReader.
The most representative implementation is JdbcPagingItemReader.
The hierarchical structure of the class is shown below.

![readerlayer](./images/7/readerlayer.png)

In addition to ItemReader, **ItemStream interface is also implemented**.

First, if you look at ItemReader, it only has `read()`.

![itemreader](./images/7/itemreader.png)

- `read()` is a method for reading data.

You can see that it is an interface that is in charge of the original task of Reader.

So what does the ItemStream interface do?
The ItemStream interface is a marker interface for **periodically saving state and restoring from that state when an error occurs**.
In other words, it plays a role in linking with the execution context of the batch process to **save the state of the ItemReader and allow it to be re-executed where it fails**.

![itemstream](./images/7/itemstream.png)

The three methods of ItemStream do the following:

- `open()` and `close()` open and close streams.
- Use `update()` to update the status of batch processing.

Developers can create the desired type of ItemReader by directly implementing the **ItemReader and ItemStream interfaces**.
However, since most of the data types in Spring Batch are already provided as ItemReaders, you will not have to implement a custom ItemReader.

> However, if your inquiry framework is Querydsl or Jooq, you may need to implement it yourself.
> It can be solved with JdbcItemReader if possible, but **JPA persistence context is not supported** so you need to implement Reader implementation yourself using HibernateItemReader.

Now, let's look at the implementation of ItemReader.
**Only implementations of Database will be discussed here**.
In addition, other Readers (File, XML, Json) are not used much in actual work, so if necessary, [Official Document](https://docs.spring.io/spring-batch/4.0.x/reference/html/ It is recommended to use it through readersAndWriters.html#flatFiles).

## 7-2. Database Reader

One of the strengths of the Spring framework is that it abstracts problems like JDBC so that developers can focus only on the business logic.

> This is called reporting service abstraction.

So Spring Batch developers have extended the JDBC capabilities of the Spring framework.

Batch jobs usually need to process large amounts of data.

> Batch applications are usually used for large-scale or large-scale data that are difficult to process in real time.

If you have a query that retrieves millions of pieces of data, you probably don't want to load all of that data into memory at once.
However, since Spring's JdbcTemplate does not support split processing (it returns the query result as it is), developers need to manually use `limit` and ` offset``. Spring Batch supports two Reader types to solve this problem. Cursors are actually a native feature of JDBC ResultSets. Whenever a ResultSet is opened, the  `next()``` method is called and the database data is returned.
This allows **Streaming** of data from the database as needed.

Paging, on the other hand, requires a bit more work.
The concept of paging is that data is retrieved from the database in chunks called pages.
In other words, it is a way to retrieve data at once by **page unit**.

Here's a pictorial comparison of Cursor and Paging:

![cursorvspaging](./images/7/cursorvspaging.png)

> In Paging, 10Row refers to PageSize.
> Other values besides 10 are possible, and 10 is used here as an example.

After establishing a connection with the database, the cursor method continuously sucks data by moving the cursor one by one.
On the other hand, in the paging method, 10 (or PageSize specified by the developer) data is retrieved at a time.

Implementations of the two methods are:

- Cursor-based ItemReader implementation
  - JdbcCursorItemReader
  - HibernateCursorItemReader
  - StoredProcedureItemReader
- Paging-based ItemReader implementation
  - JdbcPagingItemReader
  - HibernatePagingItemReader
  - JpaPagingItemReader

> IbatisReader has been removed from Spring Batch official support.
> Currently, [MyBatis Project](http://www.mybatis.org/spring/en/batch.html) is creating MyBatisReader, so please refer to it.

Since there are too many examples to cover all ItemReader examples, here, we will introduce JdbcCursorItemReader, JdbcPagingItemReader, and JpaPagingItemReader, which are representative of each Reader, with examples.

> For examples not covered here, please refer to [Official Documentation](https://docs.spring.io/spring-batch/4.0.x/reference/html/readersAndWriters.html#database) for detailed example codes. .

## 7-3. CursorItemReader

As mentioned above, CursorItemReader processes data by Streaming differently from Paging.
If you think about it easily, you can think of it as connecting a passage between the database and the application and sucking it in one by one.
Those who have written a bulletin board with JSP or Servlet can remember that data was imported one by one with `next()` using `ResultSet`.

Introducing JdbcCursorItemReader, the representative of this Cursor method.

### 7-3-1. JdbcCursorItemReader

JdbcCursorItemReader is a Cursor-based JDBC Reader implementation.
Let's take a quick look at the sample code below.

```java
@ToString
@Getter
@Setter
@NoArgsConstructor
@Entity
public class Pay {
     private static final DateTimeFormatter FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd hh:mm:ss");

     @Id
     @GeneratedValue(strategy = GenerationType.IDENTITY)
     private Long id;
     private Long amount;
     private String txName;
     private LocalDateTime txDateTime;

     public
Pay(Long amount, String txName, String txDateTime) {
        this.amount = amount;
        this.txName = txName;
        this.txDateTime = LocalDateTime.parse(txDateTime, FORMATTER);
    }

    public Pay(Long id, Long amount, String txName, String txDateTime) {
        this.id = id;
        this.amount = amount;
        this.txName = txName;
        this.txDateTime = LocalDateTime.parse(txDateTime, FORMATTER);
    }
}

@Slf4j
@RequiredArgsConstructor
@Configuration
public class JdbcCursorItemReaderJobConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final DataSource dataSource; // DataSource DI

    private static final int chunkSize = 10;

    @Bean
    public Job jdbcCursorItemReaderJob() {
        return jobBuilderFactory.get("jdbcCursorItemReaderJob")
                .start(jdbcCursorItemReaderStep())
                .build();
    }

    @Bean
    public Step jdbcCursorItemReaderStep() {
        return stepBuilderFactory.get("jdbcCursorItemReaderStep")
                .<Pay, Pay>chunk(chunkSize)
                .reader(jdbcCursorItemReader())
                .writer(jdbcCursorItemWriter())
                .build();
    }

    @Bean
    public JdbcCursorItemReader<Pay> jdbcCursorItemReader() {
        return new JdbcCursorItemReaderBuilder<Pay>()
                .fetchSize(chunkSize)
                .dataSource(dataSource)
                .rowMapper(new BeanPropertyRowMapper<>(Pay.class))
                .sql("SELECT id, amount, tx_name, tx_date_time FROM pay")
                .name("jdbcCursorItemReader")
                .build();
    }

    private ItemWriter<Pay> jdbcCursorItemWriter() {
        return list -> {
            for (Pay pay: list) {
                log.info("Current Pay={}", pay);
            }
        };
    }
}
```

The code of `Pay.java` used above is as follows.

Since the reader is not a tasklet, it cannot be done with the reader alone, so we added a simple output writer.

> **processor is not required**
> As in the example above, if there is no significant change logic for the data read by the reader, only the writer is implemented without the processor.

The settings of JdbcCursorItemReader play the following roles

_chunk
_ In `<Pay, Pay>`, **The first Pay is the type to return from the Reader**, and the **Second Pay is the type to be passed as a parameter to the Writer**.
_ If the factor value is entered as `chunkSize`, it is the range of Chunk transaction in which Reader & Writer are bound.
_ For more information on Chunk, refer to [Chapter 6](https://jojoldu.tistory.com/331).
_fetchSize
_ Indicates the amount of data to be imported from the database at one time.
* Unlike Paging, Paging divides the actual query by using `limit` and `offset`, while Cursor executes the query without division, but internally fetched data Get as much as FetchSize and bring them in one by one through `read()`.
*dataSource
* Allocate a Datasource object to use to access the database
*rowMapper
_ Mapper for mapping query results to Java instances.
_ It can be custom-created and used, but in this case, a Mapper class must be created every time, so in general, `BeanPropertyRowMapper.class` officially supported by Spring is used a lot.
_sql
_ You can use the query statement to be used as a Reader.

- name
  - Name the reader.
  - It is not the bean name, but the name to be saved in the ExecutionContext of Spring Batch.

If you look at the grammar, it looks familiar.
The reason is that JdbcItemReader has the same interface as `JdbcTemplate`, so you can use it easily without having to study separately.
If the above example is implemented as `jdbcTemplate`, it will look like the following.

```java
JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
List<Pay> payList = jdbcTemplate.query("SELECT id, amount, tx_name, tx_date_time FROM pay", new BeanPropertyRowMapper<>(Pay.class));
```

Almost no difference, right?
However, the biggest advantage of ItemReader is that **data can be streamed**.
The `read()` method takes the data one by one, passes the data to the ItemWriter, and fetches the next data again.
Through this, readers & processors & writers are executed in chunk units and committed periodically.
This is key for high-performance batch processing.

So let's test the above code.
I will register data in my own MySQL for testing.
Just run the query below.

```sql
create table pay (
   id bigint not null auto_increment,
   amount bigint,
   tx_name varchar(255),
   tx_date_time datetime,
   primary key (id)
) engine = InnoDB;

insert into pay (amount, tx_name, tx_date_time) VALUES (1000, 'trade1', '2018-09-10 00:00:00');
insert into pay (amount, tx_name, tx_date_time) VALUES (2000, 'trade2', '2018-09-10 00:00:00');
insert into pay (amount, tx_name, tx_date_time) VALUES (3000, 'trade3', '2018-09-10 00:00:00');
insert into pay (amount, tx_name, tx_date_time) VALUES (4000, 'trade4', '2018-09-10 00:00:00');
```

Now, let's run the batch.
First, let's change the Log Level to see how the query is created and executed in Reader.
Add the code below to src/main/resources/application.yml and src/test/resources/application.yml.

```yml
logging.level.org.springframework.batch: DEBUG
```

![loglevel](./images/7/loglevel.png)

그리고 배치를 실행해보시면!

![jdbccursoritemreader_result](./images/7/jdbccursoritemreader_result.png)

이렇게 등록한 데이터가 잘 조회되어 Writer에 명시한대로 데이터를 Print 하는것을 확인할 수 있습니다.

> Jpa에는 CursorItemReader가 없습니다.

### CursorItemReader의 주의 사항

CursorItemReader를 사용하실때는 Database와 SocketTimeout을 충분히 큰 값으로 설정해야만 합니다.  
**Cursor는 하나의 Connection으로 Batch가 끝날때까지 사용**되기 때문에 Batch가 끝나기전에 Database와 어플리케이션의 Connection이 먼저 끊어질수 있습니다.

그래서 **Batch 수행 시간이 오래 걸리는 경우에는 PagingItemReader를 사용하시는게 낫습니다**.  
Pagin

In the case of g, connection is established and disconnected every time a page is read, so no matter how much data there is, it can be performed without timeout and load.

## 7-4. PagingItemReader

An alternative to using Database Cursors is to run multiple queries, each query getting a portion of the result.
This processing method is called Paging.
As those of you who have implemented paging for a forum know, paging means that each query needs to specify a starting row number (`offset`) and a number of rows to return from the page (`limit`). .
Spring Batch automatically creates `offset` and `limit` according to **PageSize**.
However, it should be noted that each query is executed individually.
**It is important to sort the results when paging** as each page will run a new query.
Order by is recommended so that the order of the data results can be guaranteed.
(This will be introduced in detail below)

First, let's look at JdbcPagingItemReader.

### 7-4-1. JdbcPagingItemReader

JdbcPagingItemRedaer is a PagingItemReader that uses the same JdbcTemplate interface as JdbcCursorItemReader.
The code is shown below.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class JdbcPagingItemReaderJobConfiguration {

     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;
     private final DataSource dataSource; // DataSource DI

     private static final int chunkSize = 10;

     @Bean
     public Job jdbcPagingItemReaderJob() throws Exception {
         return jobBuilderFactory.get("jdbcPagingItemReaderJob")
                 .start(jdbcPagingItemReaderStep())
                 .build();
     }

     @Bean
     public Step jdbcPagingItemReaderStep() throws Exception {
         return stepBuilderFactory.get("jdbcPagingItemReaderStep")
                 .<Pay, Pay>chunk(chunkSize)
                 .reader(jdbcPagingItemReader())
                 .writer(jdbcPagingItemWriter())
                 .build();
     }

     @Bean
     public JdbcPagingItemReader<Pay> jdbcPagingItemReader() throws Exception {
         Map<String, Object> parameterValues = new HashMap<>();
         parameterValues. put("amount", 2000);

         return new JdbcPagingItemReaderBuilder<Pay>()
                 .pageSize(chunkSize)
                 .fetchSize(chunkSize)
                 .dataSource(dataSource)
                 .rowMapper(new BeanPropertyRowMapper<>(Pay.class))
                 .queryProvider(createQueryProvider())
                 .parameterValues(parameterValues)
                 .name("jdbcPagingItemReader")
                 .build();
     }

     private ItemWriter<Pay> jdbcPagingItemWriter() {
         return list -> {
             for (Pay pay: list) {
                 log.info("Current Pay={}", pay);
             }
         };
     }

     @Bean
     public PagingQueryProvider createQueryProvider() throws Exception {
         SqlPagingQueryProviderFactoryBean queryProvider = new SqlPagingQueryProviderFactoryBean();
         queryProvider.setDataSource(dataSource); // To choose the right PagingQueryProvider for your database
         queryProvider.setSelectClause("id, amount, tx_name, tx_date_time");
         queryProvider.setFromClause("from pay");
         queryProvider.setWhereClause("where amount >= :amount");

         Map<String, Order> sortKeys = new HashMap<>(1);
         sortKeys.put("id", Order.ASCENDING);

         queryProvider.setSortKeys(sortKeys);

         return queryProvider.getObject();
     }
}
```

If you look at the code, there is one very different setting from JdbcCursorItemReader.
Just a query (`createQueryProvider()`).
When using JdbcCursorItemReader, queries are simply created in `String` type, but in PagingItemReader, queries are created through PagingQueryProvider.
There are great reasons for doing this.

**Each database has its own strategies to support paging**.
Therefore, Spring Batch must be implemented according to the paging strategy of each database.
So, there are providers suitable for each database as follows.

![pagingprovider](./images/7/pagingprovider.png)

(Provider tailored to the paging strategy of each database)

However, this is inconvenient because you have to change the Provider code for each database.
(If you use H2 locally and use MySQL for development/operation, you won't be able to fix the Provider to one, right?)

So Spring Batch sees the Datasource setting value through **SqlPagingQueryProviderFactoryBean and automatically selects one of the providers created in the image above**.

This is an officially supported approach by Spring Batch with minimal code changes.

The values of other settings are not very different from those of JdbcCursorItemReader.

- parameterValues
  - Specifies a Map of parameter values for the query.
  - See `queryProvider.setWhereClause` for details on how to use variables.
  - The parameter variable name declared in the where clause and the parameter variable name declared in parameterValues must match.

> In the past, the parameter location was specified with `?` and each parameter value was assigned starting from 1, but it is very explicit and there is less room for mistakes.

Now, after setting this up, let's run Batch once.

![jdbcpaging_result](./images/7/jdbcpaging_result.png)

If you look at the query log, you can see that `LIMIT 10` is included.
There is no limit declaration in the written code, but it is added in the used query.
As mentioned above, this is because it is automatically added to the query according to the pageSize (fetchSize in Cursor) declared in JdbcPagingItemReader.
If there are more than 10 data to retrieve, you can appropriately retrieve as much as the next fetchSize with `offset`.

### 7-4-2. JpaPagingItemReader

Recently, many companies in Korea are increasingly using JPA.
With ORM, you can no longer view data as simple values, but as objects.
Spring Batch also officially supports JpaPagingItemReader to support JPA.

> Currently, ItemReader implementation through Querydsl, Jooq, etc. is not officially supported.
> You must create a CustomItemReader implementation.
> I will introduce this in another article.
> If you need it right away, please refer to the [official documentation](https://docs.spring.io/spring-batch/4.0.x/reference/html/readersAndWriters.html#customReader)
