#9.ItemProcessor

Chapters 7 and 8 introduced data reading and writing in chunk-oriented processing.
In this chapter, we will introduce the **processing (or processing) stage**, rather than reading and writing.
That's **ItemProcessor**.

One thing I would like to say here is that **ItemProcessor is not required**.
ItemProcessor is responsible for processing or filtering data.
This can be successfully implemented in the Writer part as well.

Nevertheless, using ItemProcessor is because it is separated as a separate step from Reader and Writer to prevent the mixing of business code.

So, in general, when adding business logic in a batch application, it is recommended to consider Processor first.
**A good way to separate each layer (read/process/write)**.

In this chapter you will learn:

- The kind of business logic that can be handled in the process step
- How to configure ItemProcessor in chunk oriented processing
- ItemProcessor implementation provided with Spring Batch

So let's get started!

## 9-1. Introduction to ItemProcessors

ItemProcessor processes/processes **individual data passed from Reader**.
Contrast with ItemWriter, which processes data grouped in ChunkSize units at once.

![process](./images/9/process.png)

In general, there are two ways to use ItemProcessor.

- conversion
  - Data read from Reader can be converted into a desired type and passed to Writer.
- filter
  - You can decide whether to pass the data passed from Reader to Writer or not.
  - If you return `null`, **this will not be passed to the Writer**

Let's learn the above two cases in turn.

## 9-2. basic usage

The ItemProcessor interface requires two generic types.

```java
package org.springframework.batch.item;

public interface ItemProcessor<I, O> {

   O process(I item) throws Exception;

}
```

_I
_ Data type to receive from ItemReader
_O
_ Data type to send to ItemWriter

The data read from the Reader passes through ItemProcessor's `process` and is delivered to the Writer.

There is only one method to implement: `process`.
Starting with Java 8, if an interface has one abstract method, you can use a **lambda expression**.
ItemProcessor also has only `process`, so you can use lambda expressions.

So, many deployments often use **anonymous classes or lambda expressions** for ItemProcessors like this:

```java
@Bean(BEAN_PREFIX + "processor")
@StepScope
public ItemProcessor<ReadType, WriteType> processor() {
     return item -> {
         item. convert();
         return item;
     };
}
```

The reasons for using anonymous classes or lambda expressions are as follows.

- There is no event code (unnecessary code), so the amount of implementation code is small
  - Can be implemented quickly.
- Since there is no fixed shape, any desired shape can be processed

However, there are also downsides.

- The amount of Batch Config code may increase because it must be included in the Batch Config class.

Due to the above disadvantages, when the amount of code increases, a separate class (implementation of ItemProcess) is used to separate the Processor.

## 9-3. conversion

The first example to look at is **transformation**.
In other words, it means converting the type read from the Reader and passing it to the Writer.

The code below reads a domain class called Teacher and passes the Name field (String type) to Wrtier.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class ProcessorConvertJobConfiguration {

     public static final String JOB_NAME = "ProcessorConvertBatch";
     public static final String BEAN_PREFIX = JOB_NAME + "_";

     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;
     private final EntityManagerFactory emf;

     @Value("${chunkSize:1000}")
     private int chunkSize;

     @Bean(JOB_NAME)
     public Job job() {
         return jobBuilderFactory.get(JOB_NAME)
                 .preventRestart()
                 .start(step())
                 .build();
     }

     @Bean(BEAN_PREFIX + "step")
     @JobScope
     public Step step() {
         return stepBuilderFactory.get(BEAN_PREFIX + "step")
                 .<Teacher, String>chunk(chunkSize)
                 .reader(reader())
                 .processor(processor())
                 .writer(writer())
                 .build();
     }

     @Bean
     public JpaPagingItemReader<Teacher> reader() {
         return new JpaPagingItemReaderBuilder<Teacher>()
                 .name(BEAN_PREFIX+"reader")
                 .entityManagerFactory(emf)
                 .pageSize(chunkSize)
                 .queryString("SELECT t FROM Teacher t")
                 .build();
     }

     @Bean
     public ItemProcessor<Teacher, String> processor() {
         return teacher -> {
             return teacher. getName();
         };
     }

     private ItemWriter<String> writer() {
         return items -> {
             for (String item : items) {
                 log.info("Teacher Name={}", item);
             }
         };
     }
}
```

In ItemProcessor, since the type to be read from Reader is `Teacher` and the type to be passed from Writer is `String`, the generic type is `<Teacher, String>`.

```java
@Bean
public ItemProcessor<Teacher, String> processor() {
     return teacher -> {
         return teacher. getName();
     };
}
```

Here, the type to be declared before ChunkSize must also follow the Reader and Writer types, so it is declared as follows.

```java
.<Teacher, String>chunk(chunkSize)
```

Try running the code above!
As shown below, you can see that `log.info("Teacher Name={}", item)` executed by Wrtier worked very well.

![convert1](./images/9/convert1.png)

In other words, you can see that the Teacher class was successfully converted to String while going through the Processor, right?

## 9-4. filter

The next example we'll look at is a **filter**.
In other words, it refers to **Determining whether or not to pass a value to the Writer in the Processor**.
The code below is an example of filtering if the id of `Teacher` is even.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class ProcessorNullJobConfiguration {

     public static final String JOB_NAME = "processorNullBatch";
     public static final String BEAN_PREFIX = JOB_NAME + "_";

     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;
     private final EntityManagerFact
     ory emf;

    @Value("${chunkSize:1000}")
    private int chunkSize;

    @Bean(JOB_NAME)
    public Job job() {
        return jobBuilderFactory.get(JOB_NAME)
                .preventRestart()
                .start(step())
                .build();
    }

    @Bean(BEAN_PREFIX + "step")
    @JobScope
    public Step step() {
        return stepBuilderFactory.get(BEAN_PREFIX + "step")
                .<Teacher, Teacher>chunk(chunkSize)
                .reader(reader())
                .processor(processor())
                .writer(writer())
                .build();
    }

    @Bean
    public JpaPagingItemReader<Teacher> reader() {
        return new JpaPagingItemReaderBuilder<Teacher>()
                .name(BEAN_PREFIX+"reader")
                .entityManagerFactory(emf)
                .pageSize(chunkSize)
                .queryString("SELECT t FROM Teacher t")
                .build();
    }

    @Bean
    public ItemProcessor<Teacher, Teacher> processor() {
        return teacher -> {

            boolean isIgnoreTarget = teacher.getId() % 2 == 0L;
            if(isIgnoreTarget){
                log.info(">>>>>>>>> Teacher name={}, isIgnoreTarget={}", teacher.getName(), isIgnoreTarget);
                return null;
            }

            return teacher;
        };
    }

    private ItemWriter<Teacher> writer() {
        return items -> {
            for (Teacher item : items) {
                log.info("Teacher Name={}", item.getName());
            }
        };
    }
}
```

In ItemProcessor, if the id is an even number, `return null;` to prevent passing it to the Writer.

```java
@Bean
public ItemProcessor<Teacher, Teacher> processor() {
     return teacher -> {

         boolean isIgnoreTarget = teacher.getId() % 2 == 0L;
         if(isIgnoreTarget){
             log.info(">>>>>>>>> Teacher name={}, isIgnoreTarget={}", teacher.getName(), isIgnoreTarget);
             return null;
         }

         return teacher;
     };
}
```

If you actually run the code, you can see that **Only odd teachers are output**.

![filter1](./images/9/filter1.png)

In this way, you can control the data to be passed to the Writer, right?

## 9-5. transaction scope

In Spring Batch, the **transaction scope is in units of chunks**.
So, if Entity is returned from Reader, **Lazy Loading between Entities is possible**.
This can be done on the Writer as well as the Processor.

Let's test if it's actually possible.

### 9-5-1. Processor

The first example is Lazy Loading on **Processor**.

The code below returns `Teacher` Entity from Reader, and Lazy Loads `Student`, the lower child of Entity, from Processor.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class TransactionProcessorJobConfiguration {

    public static final String JOB_NAME = "transactionProcessorBatch";
    public static final String BEAN_PREFIX = JOB_NAME + "_";

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final EntityManagerFactory emf;

    @Value("${chunkSize:1000}")
    private int chunkSize;

    @Bean(JOB_NAME)
    public Job job() {
        return jobBuilderFactory.get(JOB_NAME)
                .preventRestart()
                .start(step())
                .build();
    }

    @Bean(BEAN_PREFIX + "step")
    @JobScope
    public Step step() {
        return stepBuilderFactory.get(BEAN_PREFIX + "step")
                .<Teacher, ClassInformation>chunk(chunkSize)
                .reader(reader())
                .processor(processor())
                .writer(writer())
                .build();
    }


    @Bean
    public JpaPagingItemReader<Teacher> reader() {
        return new JpaPagingItemReaderBuilder<Teacher>()
                .name(BEAN_PREFIX+"reader")
                .entityManagerFactory(emf)
                .pageSize(chunkSize)
                .queryString("SELECT t FROM Teacher t")
                .build();
    }

    public ItemProcessor<Teacher, ClassInformation> processor() {
        return teacher -> new ClassInformation(teacher.getName(), teacher.getStudents().size());
    }

    private ItemWriter<ClassInformation> writer() {
        return items -> {
            log.info(">>>>>>>>>>> Item Write");
            for (ClassInformation item : items) {
                log.info("반 정보= {}", item);
            }
        };
    }
}

```

As you can see, in the Processor part, it is imported with `teacher.getStudents()`.

```java
     public ItemProcessor<Teacher, ClassInformation> processor() {
         return teacher -> new ClassInformation(teacher.getName(), teacher.getStudents().size());
     }
```

If the Processor is outside the transaction scope, an error will occur.
But if you actually run it!

![transactionProcessor](./images/9/transactionProcessor.png)

You can see that the batch ran successfully.
In other words, it was confirmed that **Processor is within the transaction scope, and Lazy Loading of Entity is possible**.

### 9-5-2. Writer

The second example is **Lazy Loading in Writer**.

The code below returns the `Teacher` Entity from the Reader, passes it directly to the Writer without going through the Processor, and Lazy Loads `Student`, the sub-children of the Entity, from the Writer.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class TransactionWriterJobConfiguration {

    public static final String JOB_NAME = "transacti
    onWriterBatch";
    public static final String BEAN_PREFIX = JOB_NAME + "_";

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final EntityManagerFactory emf;

    @Value("${chunkSize:1000}")
    private int chunkSize;

    @Bean(JOB_NAME)
    public Job job() {
        return jobBuilderFactory.get(JOB_NAME)
                .preventRestart()
                .start(step())
                .build();
    }

    @Bean(BEAN_PREFIX + "step")
    @JobScope
    public Step step() {
        return stepBuilderFactory.get(BEAN_PREFIX + "step")
                .<Teacher, Teacher>chunk(chunkSize)
                .reader(reader())
                .writer(writer())
                .build();
    }


    @Bean
    public JpaPagingItemReader<Teacher> reader() {
        return new JpaPagingItemReaderBuilder<Teacher>()
                .name(BEAN_PREFIX+"reader")
                .entityManagerFactory(emf)
                .pageSize(chunkSize)
                .queryString("SELECT t FROM Teacher t")
                .build();
    }

    private ItemWriter<Teacher> writer() {
        return items -> {
            log.info(">>>>>>>>>>> Item Write");
            for (Teacher item : items) {
                log.info("teacher={}, student Size={}", item.getName(), item.getStudents().size());
            }
        };
    }
}
```

Try running this code too!

![transactionWriter](./images/9/transactionWriter.png)

You can see that lazy loading occurs.

The two tests above confirmed that the Processor and Writer are within the transaction range and that Lazy Loading is possible.
Now, when you use JPA, you can use it more comfortably, right?

## 9-6. ItemProcessor implementation

In Spring Batch, processors for frequently used purposes are created and provided as classes in advance.
There are 3 classes in total.

\*ItemProcessorAdapter

- ValidatingItemProcessor
- CompositeItemProcessor

Ha Ji-min Recently, there are many times when Processor implementation is implemented directly, and if necessary, there are many times when it is implemented quickly with lambda expression.
So ItemProcessorAdapter and ValidatingItemProcessor are rarely used.
This is because these roles can be implemented in a custom way.
However, CompositeItemProcessor is introduced because it is sometimes necessary.

CompositeItemProcessor can be seen as a Processor that **supports chaining between ItemProcessors**.

I mentioned above that the role of Processor is transformation or filter.
But what if you need this conversion twice?
Can I convert all of them on one Processor?
Then wouldn't the role of that one processor be too big?
CompositeItemProcessor started with such a question.

The example below gets the name of `Teacher` (`getName()`) and the sentence before/after the name (`"Hello." + name + ""`) This is an example of pasting and passing it to the Writer.

```java
@Slf4j
@RequiredArgsConstructor
@Configuration
public class ProcessorCompositeJobConfiguration {

    public static final String JOB_NAME = "processorCompositeBatch";
    public static final String BEAN_PREFIX = JOB_NAME + "_";

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final EntityManagerFactory emf;

    @Value("${chunkSize:1000}")
    private int chunkSize;

    @Bean(JOB_NAME)
    public Job job() {
        return jobBuilderFactory.get(JOB_NAME)
                .preventRestart()
                .start(step())
                .build();
    }

    @Bean(BEAN_PREFIX + "step")
    @JobScope
    public Step step() {
        return stepBuilderFactory.get(BEAN_PREFIX + "step")
                .<Teacher, String>chunk(chunkSize)
                .reader(reader())
                .processor(compositeProcessor())
                .writer(writer())
                .build();
    }

    @Bean
    public JpaPagingItemReader<Teacher> reader() {
        return new JpaPagingItemReaderBuilder<Teacher>()
                .name(BEAN_PREFIX+"reader")
                .entityManagerFactory(emf)
                .pageSize(chunkSize)
                .queryString("SELECT t FROM Teacher t")
                .build();
    }

    @Bean
    public CompositeItemProcessor compositeProcessor() {
        List<ItemProcessor> delegates = new ArrayList<>(2);
        delegates.add(processor1());
        delegates.add(processor2());

        CompositeItemProcessor processor = new CompositeItemProcessor<>();

        processor.setDelegates(delegates);

        return processor;
    }

    public ItemProcessor<Teacher, String> processor1() {
        return Teacher::getName;
    }

    public ItemProcessor<String, String> processor2() {
        return name -> "안녕하세요. "+ name + "입니다.";
    }

    private ItemWriter<String> writer() {
        return items -> {
            for (String item : items) {
                log.info("Teacher Name={}", item);
            }
        };
    }
}
```

> Similar to this, it is also possible to convert to different class types.

All implementations are done by assigning `delegates`, which is an ItemProcessor List, to CompositeItemProcessor.

```java
@Bean
public CompositeItemProcessor compositeProcessor() {
     List<ItemProcessor> delegates = new ArrayList<>(2);
     delegates. add(processor1());
     delegates. add(processor2());

     CompositeItemProcessor processor = new CompositeItemProcessor<>();

     processor. setDelegates(delegates);

     return processor;
}
```

However, I cannot use generic types here.
The reason is that when using generic types, `delegates`
**All ItemProcessors included must have the same generic type**.

In this case, processor1 will have `<Teacher, String>` and processor2 will have `<String, String>`.
The current example does not use generics because you cannot use the same generic type.

If you are chaining between ItemProcessors that can use the same generic type, declaring a generic can be a safer code, right?

Now let's run this code once!

![compositeProcessor](./images/9/compositeProcessor.png)

I can confirm that it worked very well.

## Wrap-up

Reader, Writer, and Processor are all completed!
Processor doesn't have much difficulty, so I think you can take a look at it lightly.

Thank you for reading this long post!
Then see you next time. :)
