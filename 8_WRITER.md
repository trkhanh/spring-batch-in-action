#8.ItemWriter

You learned about Reader in the previous lesson.
Writer, along with Reader and Processor, are the 3 elements that make up ChunkOrientedTasklet.
The reason why I chose Writer instead of Processor here is because **Processor is optional**. A ChunkOrientedTasklet can be configured without a Processor.
On the other hand, Reader and Writer are essential elements in ChunkOrientedTasklet.

So let's deal with the Writer first.

## 8-1. Introduction to ItemWriter

ItemWriter is an **output** function used by Spring Batch.
When Spring Batch first came out, ItemWriter handled items one by one, just like ItemReader.
However, with the introduction of Spring Batch2 and chunk-based processing, ItemWriters have also undergone major changes.

After this update, ItemWriter does not write one item, but handles **item list grouped in chunk units**.
Because of this, the ItemWriter interface is slightly different from the ItemReader interface.

![itemwriter1](./images/8/itemwriter1.png)

As those who have seen [Chapter 7](https://jojoldu.tistory.com/336) know, Reader's `read()` returns one Item, while Writer's `write() ` receives an Item List as an argument.

If this is expressed graphically, it is as follows.

![write-process](./images/8/write-process.png)

- Read each item individually through the ItemReader and pass it to the ItemProcessor for processing.
- This process continues until the number of items in the chunk is processed.
- When processing is completed in chunk units, it is passed to the Writer and processed in batches as specified in the Writer.

In other words, items processed through Reader and Processor are piled up as much as Chunk units and then delivered to Writer.

> The above has already been discussed in detail in Chapter 6 Chunk Oriented Processing.

Spring Batch provides many writers to handle various output types.
As with Reader, it is difficult to cover all the contents, so we will only cover the contents related to the database.

## 8-2. Database Writer

In Java world, you use JDBC or ORM to access RDBMS.
Spring Batch provides writers for both JDBC and ORM.
Writer is the last stage of Chunk unit.
So, regarding the persistence of the database, you must **always do a flush at the end**.

For example, in the case of JPA and Hibernate that use persistence as below, `flush()` and `session.clear()` are followed in the ItemWriter implementation.

![flush1](./images/8/flush1.png)

(JpaItemWriter)

![flush2](./images/8/flush2.png)

(HibernateItemWriter)

After all items received by the writer have been processed, Spring Batch commits the current transaction.

There are three types of writers related to the database as follows.

- JdbcBatchItemWriter
- HibernateItemWriter
  \*JpaItemWriter

Among them, I will introduce JdbcBatchItemWriter and JpaItemWriter, which are used by many people.

## 8-3. JdbcBatchItemWriter

If you are not using an ORM, Writer will most likely use JdbcBatchItemWriter.
As shown in the figure below, this JdbcBatchItemWriter uses the Batch function of **JDBC and sends them to the database at once to execute queries inside the database**.

![jdbcwrite-flow](./images/8/jdbcwrite-flow.png)

The reason for this processing is to improve performance by minimizing the number of data exchanges between the application and the database.

> Refer to the official document of [JdbcTemplate.batchUpdate](https://docs.spring.io/spring/docs/3.0.0.M4/reference/html/ch12s04.html) to know the same information.

- Grouping updates into batches improves performance by reducing the number of round trips between the database and the application.

In fact, if you check `write()` of JdbcBatchItemWriter, you can see that batch processing is performed.

![jdbcwrite](./images/8/jdbcwrite.png)

Then, let's write a simple batch with `JdbcBatchItemWriter`.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class JdbcBatchItemWriterJobConfiguration {
     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;
     private final DataSource dataSource; // DataSource DI

     private static final int chunkSize = 10;

     @Bean
     public Job jdbcBatchItemWriterJob() {
         return jobBuilderFactory.get("jdbcBatchItemWriterJob")
                 .start(jdbcBatchItemWriterStep())
                 .build();
     }

     @Bean
     public Step jdbcBatchItemWriterStep() {
         return stepBuilderFactory.get("jdbcBatchItemWriterStep")
                 .<Pay, Pay>chunk(chunkSize)
                 .reader(jdbcBatchItemWriterReader())
                 .writer(jdbcBatchItemWriter())
                 .build();
     }

     @Bean
     public JdbcCursorItemReader<Pay> jdbcBatchItemWriterReader() {
         return new JdbcCursorItemReaderBuilder<Pay>()
                 .fetchSize(chunkSize)
                 .dataSource(dataSource)
                 .rowMapper(new BeanPropertyRowMapper<>(Pay.class))
                 .sql("SELECT id, amount, tx_name, tx_date_time FROM pay")
                 .name("jdbcBatchItemWriter")
                 .build();
     }

     /**
      * A writer that outputs the data passed from the reader one by one
      */
     @Bean // Required when using beanMapped()
     public JdbcBatchItemWriter<Pay> jdbcBatchItemWriter() {
         return new JdbcBatchItemWriterBuilder<Pay>()
                 .dataSource(dataSource)
                 .sql("insert into pay2(amount, tx_name, tx_date_time) values (:amount, :txName, :txDateTime)")
                 .beanMapped()
                 .build();
     }
}

```

Most of the code is similar to the code used in the [Reader chapter](https://jojoldu.tistory.com/336).
Only the code using JdbcBatchItemWriterBuilder is slightly different, so I'll explain in a bit more detail.

JdbcBatchItemWriterBuilder has the following settings

| Property      | Parameter Type | Description                                                                                                                                                |
| ------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| assertUpdates | boolean        | Sets whether to throw an exception if at least one item does not update or delete a row. The default is `true`. Exception:`EmptyResultDataAccessException` |
| columnMapped  | None           | Map Insert SQL Values based on Key,Value (ex: `Map<String, Object>`)                                                                                       |
| beanMapped    | None           | Match the Values of Insert SQL based on Pojo                                                                                                               |
| Ping          |

You might be wondering about the difference between `columnMapped` and `beanMapped` here.
For example, the example above is written with ` beanMapped``. If you change the above example to  `columnMapped```, the code becomes the following.

```java

new JdbcBatchItemWriterBuilder<Map<String, Object>>() // use Map
.columnMapped()
.dataSource(this.dataSource)
.sql("insert into pay2(amount, tx_name, tx_date_time) values (:amount, :txName, :txDateTime)")
.build();

```

The difference is simple. Whether the type passed from Reader to Writer is `Map<String, Object>` or Pojo type such as `Pay.class`.

Another thing you might be curious about is `values(:field)`.
In the case of this value, the value is assigned by being mapped to **Dto's getter or Map's key**.

In addition, there is one thing to note in the setting of `JdbcBatchItemWriter`, not JdbcBatchItemWriterBuilder.

- The generic type of JdbcBatchItemWriter is the type of the value passed from **Reader**.

This part is often misunderstood by those new to Spring Batch.
As shown in the code above, **Writer who put data into the Pay2 table, but the declared generic type is the Pay class handed over from Reader/Processor**.

In addition, there is an additional method that you should know about: `afterPropertiesSet`.
This method is a method of the `InitializingBean` interface.
ItemWriter implementations such as JdbcBatchItemWriter and JpaItemWriter all implement the `InitializingBean` interface.
What `afterPropertiesSet` does here is to check whether the necessary values for each Writer are properly set.

![afterpropertiesset1](./images/8/afterpropertiesset1.png)

If you create a Writer and execute the above method directly under it, you can clearly recognize which value is missing, so it is a commonly used option.

## 8-4. JpaItemWriter

The second writer to look at is `JpaItemWriter` that can use ORM.
If the data to be passed to the Writer is an Entity class, you can use JpaItemWriter.

Let's go straight to the sample code.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class JpaItemWriterJobConfiguration {
     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;
     private final EntityManagerFactory entityManagerFactory;

     private static final int chunkSize = 10;

     @Bean
     public Job jpaItemWriterJob() {
         return jobBuilderFactory.get("jpaItemWriterJob")
                 .start(jpaItemWriterStep())
                 .build();
     }

     @Bean
     public Step jpaItemWriterStep() {
         return stepBuilderFactory.get("jpaItemWriterStep")
                 .<Pay, Pay2>chunk(chunkSize)
                 .reader(jpaItemWriterReader())
                 .processor(jpaItemProcessor())
                 .writer(jpaItemWriter())
                 .build();
     }

     @Bean
     public JpaPagingItemReader<Pay> jpaItemWriterReader() {
         return new JpaPagingItemReaderBuilder<Pay>()
                 .name("jpaItemWriterReader")
                 .entityManagerFactory(entityManagerFactory)
                 .pageSize(chunkSize)
                 .queryString("SELECT p FROM Pay p")
                 .build();
     }

     @Bean
     public ItemProcessor<Pay, Pay2> jpaItemProcessor() {
         return pay -> new Pay2(pay.getAmount(), pay.getTxName(), pay.getTxDateTime());
     }

     @Bean
     public JpaItemWriter<Pay2> jpaItemWriter() {
         JpaItemWriter<Pay2> jpaItemWriter = new JpaItemWriter<>();
         jpaItemWriter.setEntityManagerFactory(entityManagerFactory);
         return jpaItemWriter;
     }
}
```

Since JpaItemWriter uses JPA, EntityManager must be assigned to manage persistence.

> In general, if `spring-boot-starter-data-jpa` is registered as a dependency, Entity Manager is automatically created as a bean, and only DI code needs to be added.

Instead, the only value that needs to be set is EntityManager.
Compared to JdbcBatchItemWriter, the required value is Entity Manager only, so the fact that there are fewer elements to check is an advantage rather than an advantage.

![afterpropertiesset2](./images/8/afterpropertiesset2.png)

(required value checking method `afterPropertiesSet`)

If you `set` only the EntityManager, all settings are finished.

One thing here that is different from JdbcBatchItemWriter is the addition of a processor.
The reason is to read the Pay Entity and pass the Pay2 Entity to the Writer.

> Processor is needed when data read from Reader needs to be processed.

Unlike JdbcBatchItemWriter, JpaItemWriter reflects the passed Entity to the database.

That is, JpaItemWriter must receive the **Entity class as a generic type**.
In the case of JdbcBatchItemWriter, there is no problem as the query designated as `sql` is executed even if the DTO class is received, but JpaItemWriter reflects the transferred item to the table with ```entityManger.

![dowrite](./images/8/dowrite.png)

(`JpaItemWriter.doWrite()`)

With this setting, the usage of JpaItemWriter is over.
If you actually run it, you can see that the result comes out normally.

![jpaitemwriter-result](./images/8/jpaitemwriter-result.png)

## 8-5. Custom ItemWriter

Unlike Reader, Writer requires many custom implementations.

> Of course, the Reader may also need to be implemented as a Custom depending on which framework for inquiry is used.
> For example, you can create an ItemReader based on Querydsl or an ItemReader based on Jooq.

For example:

- When data read from Reader needs to be transferred to an external API using RestTemplate
- When you need to put a value into a singleton object for temporary storage and comparison
- When you need to save multiple entities at the same time

And so on, there are several situations.
If you want to use a writer that is not officially supported by Spring Batch like this, you can **implement the ItemWriter interface**.

Below is an example of creating a Writer that outputs the data received from the processor as `System.out.println`.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class CustomItemWriterJobConfiguration {
     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;
     private final EntityManagerFactory entityManagerFact
     ory;

    private static final int chunkSize = 10;

    @Bean
    public Job customItemWriterJob() {
        return jobBuilderFactory.get("customItemWriterJob")
                .start(customItemWriterStep())
                .build();
    }

    @Bean
    public Step customItemWriterStep() {
        return stepBuilderFactory.get("customItemWriterStep")
                .<Pay, Pay2>chunk(chunkSize)
                .reader(customItemWriterReader())
                .processor(customItemWriterProcessor())
                .writer(customItemWriter())
                .build();
    }

    @Bean
    public JpaPagingItemReader<Pay> customItemWriterReader() {
        return new JpaPagingItemReaderBuilder<Pay>()
                .name("customItemWriterReader")
                .entityManagerFactory(entityManagerFactory)
                .pageSize(chunkSize)
                .queryString("SELECT p FROM Pay p")
                .build();
    }

    @Bean
    public ItemProcessor<Pay, Pay2> customItemWriterProcessor() {
        return pay -> new Pay2(pay.getAmount(), pay.getTxName(), pay.getTxDateTime());
    }

    @Bean
    public ItemWriter<Pay2> customItemWriter() {
        return new ItemWriter<Pay2>() {
            @Override
            public void write(List<? extends Pay2> items) throws Exception {
                for (Pay2 item : items) {
                    System.out.println(item);
                }
            }
        };
    }
}
```

As you can see, if you only `write()`, `@Override`, implementation is created.
The code above can be used in Java 7 or lower, but if you are using Java 8 or higher, you can implement it more neatly by using a lambda expression as shown below.

```java
    @Bean
    public ItemWriter<Pay2> customItemWriter() {
        return items -> {
            for (Pay2 item : items) {
                System.out.println(item);
            }
        };
    }
```

## 8-6. caution

When using ItemWriter, there are times when you want to pass a List from Processor to Writer.
At this time, the problem cannot be solved by declaring the generic of ItemWriter as List.
I wrote in detail how to solve it in the link below, so it would be nice to refer to it.

- [When you want to send List-type Items to the Writer](https://jojoldu.tistory.com/140)
