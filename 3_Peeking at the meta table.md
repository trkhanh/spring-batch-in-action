# 3. Spring Batch meta table peek

This time, we will take a closer look at Spring Batch's meta table.

> All the code I worked on is on [Github](https://github.com/jojoldu/spring-batch-in-action), so you can refer to it.

Last time, I showed you a little bit of Spring Batch's meta table.

![batchdomain](./images/3/batchdomain.png)

We will introduce the role of these meta tables and what they contain through practice.

## 3-1. BATCH_JOB_INSTANCE

The first thing to look at is `BATCH_JOB_INSTANCE`.
If you do a query in the local MySQL, you will find 1 ROW as shown below.

![meta1](./images/3/meta1.png)

- `JOB_INSTANCE_ID`
  - PK from table `BATCH_JOB_INSTANCE`
- `JOB_NAME`
  - Executed Batch Job Name

You should see the simpleJob you just ran.
The BATCH_JOB_INSTANCE table is a table created according to the **Job Parameters**.
You may be unfamiliar with this Job Parameter.
To put it simply, **Parameters that can be received from the outside when Spring Batch is running**.

For example, if you pass a specific date as a Job Parameter, **In Spring Batch, you can search/process/enter data with that date**.

Even for the same batch job, if the job parameters are different, it is recorded in `Batch_JOB_INSTANCE`, and **if the job parameters are the same, it is not recorded**.

Shall we check it out?
Modify the simpleJob code as below.

![simpleJob10](./images/3/simpleJob10.png)

- Don't forget `@JobScope`.

```java
@Slf4j // lombok annotation for using log
@RequiredArgsConstructor // lombok annotation for constructor DI
@Configuration
public class SimpleJobConfiguration {
     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;

     @Bean
     public Job simpleJob() {
         return jobBuilderFactory.get("simpleJob")
                 .start(simpleStep1(null))
                 .build();
     }

     @Bean
     @JobScope
     public Step simpleStep1(@Value("#{jobParameters[requestDate]}") String requestDate) {
         return stepBuilderFactory.get("simpleStep1")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> This is Step1");
                     log.info(">>>>> requestDate = {}", requestDate);
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }
}
```

Only the code marked in red has been added, so you can change it as it is.

> Batch Job Parameters will be introduced in detail in later chapters.
> This is a sample code for understanding the meta table, so you only need to understand it to the extent that you can use Job Parameters in this way. :)

The changed code is a function that additionally outputs the value received as a Job Parameter to the log.

Now, let's run Batch by inserting Job Parameters.
Just like before, click your IDE Run Environment tab to go to the settings window.

![simpleJob11](./images/3/simpleJob11.png)

Enter `requestDate=20180805` in **Program arguments** as shown below.

![simpleJob12](./images/3/simpleJob12.png)

Save and try again.
then!
You can see that the Job Parameter is delivered very well and the log is taken.

![simpleJob13](./images/3/simpleJob13.png)

Let's look at BATCH_JOB_INSTANCE again.

![meta1-1](./images/3/meta1-1.png)

A new Job Instance has been added!
Now, let's see if a new Job Parameter is not created if the real Job Parameter is the same.
Run Batch in the IDE again with the same parameters.

![meta1-2](./images/3/meta1-2.png)

There is an error message along with an exception called **JobInstanceAlreadyCompleteException**.

```
A job instance already exists and is complete for parameters={requestDate=20180805}. If you want to run this job again, change the parameters.
```

Doesn't it say you can't run a job with the same parameters?
Let's change the parameters and run it.
This time, `requestDate=20180806`.

![simpleJob14](./images/3/simpleJob14.png)

It was executed normally, and the `BATCH_JOB_INSTANCE` table was also added normally!

![meta1-3](./images/3/meta1-3.png)

In other words, the same Job is created in `BATCH_JOB_INSTANCE` every time the Job Parameter changes, and **there cannot be multiple identical Job Parameters**.

> I think the table name is job instance, which is a really appropriate name.
> In a way, it feels similar to creating multiple instances through a Java class.

## 3-2. BATCH_JOB_EXECUTION

The next thing to look at is the `BATCH_JOB_EXECUTION` table.

![meta2](./images/3/meta2.png)

If you look at the JOB_EXECUTION table, there are 3 rows.
These are the three execution data: the simpleJob we executed with no parameters, the simpleJob we executed with the parameter `requestDate=20180805`, and the simpleJob we executed with the parameter ``requestDate=20180806`.

Now, looking at it like this, it doesn't look much different from JOB_INSTANCE, right?
**JOB_EXECUTION and JOB_INSTANCE are parent-child relationships**.
JOB_EXECUTION has all success/failure history of its parent **JOB_INSTACNE**.
Let's try it out.

Let's change the simpleJob code as below.

![meta2-job1](./images/3/meta2-job1.png)

```java
@Slf4j // lombok annotation for using log
@RequiredArgsConstructor // lombok annotation for constructor DI
@Configuration
public class SimpleJobConfiguration {
     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;

     @Bean
     public Job simpleJob() {
         return jobBuilderFactory.get("simpleJob")
                 .start(simpleStep1(null))
                 .next(simpleStep2(null))
                 .build();
     }

     @Bean
     @JobScope
     public Step simpleStep1(@Value("#{jobParameters[requestDate]}") String requestDate) {
         return stepBuilderFactory.get("simpleStep1")
                 .tasklet((contribution, chunkContext) -> {
                     throw new IllegalArgumentException("Step1 failed.");
                 })
                 .build();
     }

     @Bean
     @JobScope
     public Step simpleStep2(@Value("#{jobParameters[requestDate]}") String requestDate) {
         return stepBuilderFactory.get("simpleStep2")
         tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> This is Step2");
                     log.info(">>>>> requestDate = {}", requestDate);
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }
}
```

This time, change the Job Parameter to `requestDate=20180807`.

![meta2-job2](./images/3/meta2-job2.png)

Try running it now!

![meta2-job3](./images/3/meta2-job3.png)

You can see that the Batch Job failed with the Exception we threw.
Now, let's see how the BATCH_JOB_EXECUTION table has changed.

![meta2-job4](./images/3/meta2-job4.png)

You can see that the EXECUTION holding the 4th JOB_INSTACNE as FK is FAILED.
Now, let's modify the code to make the JOB successful.

![meta2-job5](./images/3/meta2-job5.png)

If you run Batch again with the changed code!

![meta2-job6](./images/3/meta2-job6.png)

JOB has been done successfully.
Shall we see the table?

![meta2-job7](./images/3/meta2-job7.png)

This is the crucial difference from JOB_INSTANCE.
If you look at the JOB_INSTANCE_ID column of BATCH_JOB_EXECUTION, you can see two rows with the same ID (`4`).
The first ROW of them has STATUS FAILED, but the second ROW is COMPLETED.

You can see that BATCH_JOB_INSTACNE (id=``4`) created with the Job Parameter `requestDate=20180807``` was executed twice, the first failed, and the second succeeded.

What's interesting here is that **I ran it twice with the same Job Parameter, but there was no error saying that it was executed with the same parameter**.
You can know that Spring Batch cannot be re-executed only when there is a successful record with the same Job Parameter.

Now, let's recap.

## 3-3. JOB, JOB_INSTANCE, JOB_EXECUTION

The relationship between the two tables above and the Spring Batch Job we created is as follows.

![jobschema](./images/3/jobschema.png)

The Job here refers to the Spring Batch Job we created.

In addition to the two tables above, there are more Job-related tables.
For example, the `BATCH_JOB_EXECUTION_PARAM` table contains the job parameters entered when the BATCH_JOB_EXECUTION table was created.

![meta3](./images/3/meta3.png)

In addition, various meta tables exist.
Each table will be introduced additionally whenever necessary in the future process.

> In addition to JOB_EXECUTION, there are several tables related to **STEP_EXECUTION**.
> This part is too big to cover now, so I will introduce it in detail in the next **Spring Batch Retry/SKIP Strategy**.

The overall story of Spring Batch was divided into three parts.
Now, I will introduce practical Spring Batch examples and codes in earnest.

## 3-4. Spring Batch Test code?

As you can see from my previous [Spring Batch articles](http://jojoldu.tistory.com/search/batch), I always wrote my Spring Batch examples as test code.
However, there were drawbacks to this method.
**I've seen a lot of people using Spring Batch in the development/production environment struggle with Batch Job Instance Context problems**.
We recommend that you refrain from testing code using H2 until you adapt to Spring Batch.
The test code using H2 will be shown as soon as possible.
In the beginning, while using MySQL, **We will proceed to adapt as much as possible to Spring Batch with meta table information remaining**.
