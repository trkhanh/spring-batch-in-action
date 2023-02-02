# 4. Spring Batch Job Flow

Now, from this time, let's learn the contents of Spring Batch that can be used in practice.

> All the code I worked on is on [Github](https://github.com/jojoldu/spring-batch-in-action), so you can refer to it.

I mentioned earlier that there are steps to configure a Spring Batch job.
Step plays the role of **actual batch operation**.
If you look at the code I wrote before, there is almost no code in Job.

The function that actually handles the batch business logic (`ex: log.info()`) is implemented in the Step.
In this way, you can think of a step as a place that includes all functions and settings that you actually want to process as a batch.

Since it contains batch processing contents, it is necessary to control the order or processing flow between **Steps** in the Job.
This time, let's look at how to manage multiple Steps.

## 4-1. Next

The first thing to learn is **Next**.

Next, while working on `simpleJob` earlier, we dealt with it a bit, right?
Let's write a sample code.
The batch to be created this time will be made with `StepNextJobConfiguration.java`.

```java
@Slf4j
@Configuration
@RequiredArgsConstructor
public class StepNextJobConfiguration {

     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;

     @Bean
     public Job stepNextJob() {
         return jobBuilderFactory.get("stepNextJob")
                 .start(step1())
                 .next(step2())
                 .next(step3())
                 .build();
     }

     @Bean
     public Step step1() {
         return stepBuilderFactory.get("step1")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> This is Step1");
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }

     @Bean
     public Step step2() {
         return stepBuilderFactory.get("step2")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> This is Step2");
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }

     @Bean
     public Step step3() {
         return stepBuilderFactory.get("step3")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> This is Step3");
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }
}
```

As you can see, `next()` is **used when sequentially connecting Steps**.
When executing step1 -> step2 -> step3 one by one, `next()` is a good method.

Now, shall we run it to see if it is called sequentially?
This time, after changing the Job Parameter to `version=1`

![next1](./images/4/next1.png)

Try running it!

![next2](./images/4/next2.png)

The stepNextJob batch was executed, but the **existing simpleJob was also executed**.

We only want to run the batch of stepNextJob we just created, but when all the batches created run, we won't be able to use it, right?
So a little bit so that only the specified batches are performed? Let's change the settings.

### 1. Make sure that only the specified Batch Job is executed.

Add the code below to `src/main/resources/application.yml` in your project.

```yaml
spring.batch.job.names: ${job.name:NONE}
```

![jobname1](./images/4/jobname1.png)

What the added options do is simple.
When Spring Batch is executed, if `job.name` value is passed as **Program arguments, only jobs that match the corresponding value will be executed**.
Here, if you look at `${job.name:NONE}`, `job.name` is on the left and `NONE` is on the right, with `:` in between. I have this.
The meaning of this code is that ** if `job.name`** is present, assign a value to `job.name`**, if not ** assign `NONE`\*\* ** means to do.
An important thing is! If `NONE` is assigned to `spring.batch.job.names`, it means that **no batches will be executed**.
In other words, it is a role that prevents all batches from being executed when **value does not exist\*\*.

Now, let's modify the execution environment of the IDE again so that `job.name` mentioned above is passed as Program arguments during batch execution.

In the execution environment of the IDE, enter the following code in the Program arguments item where we modified the Job Parameter.

```java
--job.name=stepNextJob
```

![jobname2](./images/4/jobname2.png)

Just add this.
(The **version next to it should be changed to 2 because number 1 has already been executed**.)

Now, save and run it.
Since version = 2 has been changed, duplication of Job Instances will not occur, so only `stepNextJob` should be executed normally, right?

Try it once!

![jobname3](./images/4/jobname3.png)

Only the specified `stepNextJob` was performed!
Now, you can run only the necessary Jobs by changing the values, right?

> In an actual operating environment, execute batch as `java -jar batch-application.jar --job.name=simpleJob `.
> These operational environments will be introduced later in the series.

## 4-2. Flow control by condition (Flow)

Now, I found out that Next sequentially controls the order of Steps.
The important thing here is that **If an error occurs in the preceding step, the rest of the following steps will not be executed**.

However, depending on the situation, there are times when it is necessary to proceed with **Step B when it is normal, and Step C when there is an error**.

![conditional1](./images/4/conditional1.png)

In preparation for this case, Spring Batch Job can use Steps for each condition.
Let's take a look by creating a new class `StepNextConditionalJobConfiguration`.

```java
@Slf4j
@Configuration
@RequiredArgsConstructor
public class StepNextConditionalJobConfiguration {

     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;

     @Bean
     public Job stepNextConditionalJob() {
         return jobBuilderFactory.get("stepNextConditionalJob")
                 .start(conditionalJobStep1())
                     .on("FAILED") // In case of FAILED
                     .to(conditionalJobStep3()) // Go to step3.
                     .on("*") // regardless of the result of step3
                     .end() // Flow ends when moving to step3.
                 .from(conditionalJobStep1()) // from step1
                     .on("*") // in all cases except FAILED
                     .to(conditionalJobStep2()) // Move to step2.
                     .next(conditionalJobStep3()) // If step2 ends normally, move to step3
                     do.
                     .on("*") // regardless of the result of step3
                     .end() // Flow ends when moving to step3.
                 .end() // end the job
                 .build();
     }

     @Bean
     public Step conditionalJobStep1() {
         return stepBuilderFactory.get("step1")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> This is stepNextConditionalJob Step1");

                     /**
                         Set ExitStatus to FAILED.
                         The flow proceeds by looking at the corresponding status.
                     **/
                     contribution.setExitStatus(ExitStatus.FAILED);

                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }

     @Bean
     public Step conditionalJobStep2() {
         return stepBuilderFactory.get("conditionalJobStep2")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> This is stepNextConditionalJob Step2");
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }

     @Bean
     public Step conditionalJobStep3() {
         return stepBuilderFactory.get("conditionalJobStep3")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> This is stepNextConditionalJob Step3");
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }
}
```

The above code scenario differs depending on whether step1 fails or succeeds.

- step1 failure scenario: step1 -> step3
- step1 success scenario: step1 -> step2 -> step3

The code that manages this whole flow is right below.

![from1](./images/4/from1.png)

- `.on()`
  - Specify **ExitStatus** to catch
  - In case of `*`, all ExitStatus are specified.
- ``to()`
  - Specify the step to move to next
- `from()`
  - acts as a kind of **event listener**
  - Look at the status value and call the `step` included in `to()` if the status matches.
  - In the state where event catch in step1 is FAILED, `from` must be used to **additionally catch event**
- `end()`
  - end has two ends, one that returns FlowBuilder and the other that terminates FlowBuilder.
  - The end after `on("*")` returns the FlowBuilder.
  - The end before `build()` ends FlowBuilder.
  - When using end that returns FlowBuilder, you can continue `from`

The important point here is that the status value that `on` catches is ExitStatus, not BatchStatus.
So, if you need to adjust the status value for branch processing, you need to adjust ExitStatus.
The code to adjust is below.

![from2](./images/4/from2.png)

You can change the value of `contribution.setExitStatus` by writing branch logic according to the situation you want.
Here, we will test the step1 -> step3 flow by generating FAILED first.

Now, once the code is written like this, let's run it.

![from3](./images/4/from3.png)

**You can see that only step1 and step3 have been executed**!
Step2 was ignored and executed because of ExitStatus.FAILED?
Now, let's modify the code a little and see if step1->step2->step3 becomes.

![from4](./images/4/from4.png)

Comment it out and try again!

![from5](./images/4/from5.png)

You can see that step1->step2->step3 is executed in sequence as a normal flow!
Now, if you need to call a different step for each condition, you can do it very easily, right?

### Extra 2. Batch Status vs. Exit Status

Although briefly mentioned in the discussion of conditional flow control above, it is important to know the difference between **BatchStatus and ExitStatus**.

BatchStatus is an Enum used when **recording the execution result of Job or Step in Spring**.
The values used for BatchStatus are COMPLETED, STARTING, STARTED, STOPPING, STOPPED, FAILED, ABANDONED, and UNKNOWN.
Most of the values can be interpreted and understood as the same meaning as the word.

![batchstatus](./images/4/batchstatus.png)

for example,

```xml
.on("FAILED").to(stepB())
```

In the code above, you can think of what the `on` method refers to as BatchStatus, but the actual referenced value is **Step's ExitStatus**.

ExitStatus refers to the **status after execution of Step**.

![exitstatus](./images/4/exitstatus.png)

(`ExitStatus` is not an Enum.)

To make the above example (`.on("FAILED").to(stepB())`) easier, it means **go to StepB if exitCode ends with FAILED**.
Spring Batch **Basically, ExitStatus exitCode is set to be the same as BatchStatus of Step**.
But what if you need your own custom exitCode?
(That is, a situation that must be different from BatchStatus.)

Let's look at example code.

```xml
.start(step1())
     .on("FAILED")
     .end()
.from(step1())
     .on("COMPLETED WITH SKIPS")
     .to(errorPrint1())
     .end()
.from(step1())
     .on("*")
     .to(step2())
     .end()
```

The execution result of step 1 above can be the following three.

- Step1 fails, and Job also fails.
- Step1 is executed successfully, so step2 is executed.
- Step1 completes successfully, and ends with the exit code of `COMPLETED WITH SKIPS`.

`COMPLETED WITH SKIPS` in the code above is not present in ExitStatus.
You need separate logic to return exitCode `COMPLETED WITH SKIPS` in order to be handled as you want.

```java
public class SkipCheckingListener extends StepExecutionListenerSupport {

     public ExitStatus afterStep(StepExecution stepExecution) {
         String exitCode = stepExecution.getExitStatus().getExitCode();
         if (!exitCode.equals(ExitStatus.FAILED.getExitCode()) &&
               stepExecution.getSkipCount() > 0) {
             return new ExitStatus("COMPLETED WITH SKIPS");
         }
         else {
             return null;
         }
     }
}
```

To explain the code above, StepExecutionListener first checks whether the step was successfully executed, **If the number of skips in StepExecution is greater than 0**, it returns ExitStatus with an exitCode of `COMPLETED WITH SKIPS` \*\*\*\* do.

## 4-3. Decide

Now, in (4-2), we looked at how to move to different Steps according to the results of Steps.
This time in a different way
Let's look at processing.
The above method has two problems.

- Steps will have more than one role.
  - In addition to the logic to be processed by the actual step, ExitStatus operation is required for branch processing.
- Difficulty handling various branch logic
  - In order to customize ExitStatus, there is a hassle such as creating a listener and registering it in the Job Flow.

It would be convenient to have a type that can handle various branching while clearly only responsible for branching the Flow between Steps, right?
So, in Spring Batch, there is a type that only handles branching in the Flow of Steps.
It is called JobExecutionDecider, and let's create a sample code using it.
The class name will be `DeciderJobConfiguration`.

```java
@Slf4j
@Configuration
@RequiredArgsConstructor
public class DeciderJobConfiguration {
     private final JobBuilderFactory jobBuilderFactory;
     private final StepBuilderFactory stepBuilderFactory;

     @Bean
     public Job deciderJob() {
         return jobBuilderFactory.get("deciderJob")
                 .start(startStep())
                 .next(decider()) // odd | even division
                 .from(decider()) // the state of the decider
                     .on("ODD") // if ODD
                     .to(oddStep()) // Go to oddStep.
                 .from(decider()) // the state of the decider
                     .on("EVEN") // if ODD
                     .to(evenStep()) // Go to evenStep.
                 .end() // end builder
                 .build();
     }

     @Bean
     public Step startStep() {
         return stepBuilderFactory.get("startStep")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> Start!");
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }

     @Bean
     public Step evenStep() {
         return stepBuilderFactory.get("evenStep")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> even number");
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }

     @Bean
     public Step oddStep() {
         return stepBuilderFactory.get("oddStep")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> odd.");
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }

     @Bean
     public JobExecutionDecider decider() {
         return new OddDecider();
     }

     public static class OddDecider implements JobExecutionDecider {

         @Override
         public FlowExecutionStatus decide(JobExecution jobExecution, StepExecution stepExecution) {
             Random rand = new Random();

             int randomNumber = rand.nextInt(50) + 1;
             log.info("Random number: {}", randomNumber);

             if(randomNumber % 2 == 0) {
                 return new FlowExecutionStatus("EVEN");
             } else {
                 return new FlowExecutionStatus("ODD");
             }
         }
     }
}

```

The flow of this batch is as follows.

- startStep -> Distinguish odd or even number in oddDecider -> proceed to oddStep or evenStep

The logic to insert the decider between flows is as follows.

![decider1](./images/4/decider1.png)

- `start()`
  - Start the first step of Job Flow.
- `next()`
  - Run `decider` after `startStep`.
- `from()`
  - As in 4-2, from acts as an event listener.
  - Check the state value of the decider and call the `step` included in `to()` if the state matches.

The code is not very different from 4-2, so you won't have any difficulty understanding it.

As you can see, all branch logic is handled by `OddDecider`.
No matter how complex branching logic is required, it is now possible to proceed with **roles and responsibilities** clearly separated from Steps.

Let's take a look at the Decider implementation.

![decider2](./images/4/decider2.png)

An OddDecider that implements the JobExecutionDecider interface.
Here, we randomly generate a number and return different statuses depending on whether it is odd or even.
Note that the state is managed with `FlowExecutionStatus` rather than ExitStatus because it is not processed as a step.

It is very easy to create and return states called EVEN and ODD, and you can see that they are used in `from().on()`.
Now, let's try this code.

![decider3](./images/4/decider3.png)

![decider4](./images/4/decider4.png)

If you run it several times, you can see that different steps (oddStep, evenStep) are executed with odd/even numbers!

## Wrap-up

How was it?
Wasn't it difficult?
It would be nice if it gave you a hint on how to configure Job Flow when configuring Spring Batch Job.
Next time, we will talk about `Scope`, the most important concept of Spring Batch.
Thank you for reading this long post :)

## Reference

- [VM Arguments, Program arguments](https://stackoverflow.com/a/37439625)
