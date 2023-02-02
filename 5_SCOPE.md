# 5. Spring Batch Scope & Job Parameter

In this time, we will learn about Spring Batch scope.
Scope here refers to `@StepScope` and `@JobScope`.
Let's see what these annotations we use unconsciously actually do.
And let's learn the **Job Parameter** that can't be separated from these two.

## 5-1. JobParameters and Scopes

In the case of Spring Batch, it receives parameters from outside or inside and supports them so that they can be used in multiple batch components.
These parameters are called **Job Parameters**.
In order to use Job Parameters, you always have to declare a scope dedicated to Spring Batch.
There are two main types, `@StepScope` and `@JobScope`.
It is simple to use, and you can use it by declaring it in SpEL as shown below.

```java
@Value("#{jobParameters[parameter name]}")
```

> In addition to ` jobParameters``,  `jobExecutionContext`and`stepExecutionContext` can be used as SpEL. @JobScope cannot use ```stepExecutionContext```, only ```jobParameters` and ``jobExecutionContext`.

The sample code used in each scope is as follows.

**JobScope**

![sample-jobscope](./images/5/sample-jobscope.png)

**StepScope**

![sample-stepscope](./images/5/sample-stepscope.png)

**@JobScope can be used in Step declarations**, and **@StepScope can be used in Tasklets, ItemReaders, ItemWriters, and ItemProcessors**.

There are `Double`, `Long`, `Date`, and `String` that can be used as the current Job Parameter type.
Unfortunately, there are no `LocalDate` and `LocalDateTime`, so you have to receive them as `String` and convert them before using them.

If you look at the example code, the calling side is assigning `null`.
This is possible because **Job Parameter allocation is not done at the time of application execution**.
Now, let's go into a bit more detail about what this is all about.

## 5-2. Introducing @StepScope & @JobScope

Spring Batch supports very special bean scopes, `@StepScope` and `@JobScope`.
As you know, the **default scope of Spring Bean is singleton**.
However, if you use `@StepScope` in Spring Batch components (Tasklet, ItemReader, ItemWriter, ItemProcessor, etc.) as shown below,

![stepscope1](./images/5/stepscope1.png)

Spring Batch creates the corresponding component as a Spring bean** at the time of execution of the specified **Step through the Spring container**.
Likewise, `@JobScope` creates a bean at **Job execution time\*\*.
In other words, it delays the creation of the bean to the point at which the specified scope is executed.

> In a way, it can be similar to MVC's request scope.
> Just as a request scope is created when a request comes and deleted when a response is returned, JobScope and StepScope are also created/deleted when the Job is executed and finished, and when the Step is executed and finished.

There are two main advantages of delaying the bean creation time to the step or job execution time, not the application execution time.

First, **Late Binding of JobParameter** is possible.
Job parameters can be assigned at the StepContext or JobExecutionContext level.
Job parameters can be assigned at the **business logic processing stage**, such as a controller or service, even when the application is not running.
This part will be explained in more detail with an example below.

Second, it is useful when using the same component in parallel or concurrently.
Let's assume there is a Tasklet inside a Step, and this Tasklet has a member variable and a logic that changes this member variable.
In this case, if Steps are executed in parallel without `@StepScope`, **one Tasklet will be placed in different Steps and the state will be changed randomly**.
However, if you have `@StepScope` **Since each Step creates and manages a separate Tasklet, there is nothing to invade each other's state**.

## 5-3. Job Parameter Misunderstanding

Job Parameters are available through `@Value`.
As a result, there may be various misunderstandings.
Job Parameters can be called at the time of creation of Batch component beans such as Step, Tasklet, Reader, etc., but it is possible only when **Scope Bean is created**.
In other words, it can be used because Job Parameters are created** only when **`@StepScope` and `@JobScope` beans are created.

For example, let's create a bean directly in a class instead of creating a bean through a method as shown below.
**Remove** `@Bean` and ` @Value("#{jobParameters[parameter name]}")`` from the code of Job and Step and add  `SimpleJobTasklet`` Receive it as constructor DI.

> You can also use `@Autowired`.

![jobparameter1](./images/5/jobparameter1.png)

And `SimpleJobTasklet` is created as a bean whose **Scope is Step** with `@Component` and ` @StepScope`` as shown below. In this state, assign  `@Value("#{jobParameters[parameter name]}``` as a member variable of Tasklet.

![jobparameter2](./images/5/jobparameter2.png)

Even if JobParameter is not assigned as a method parameter, JobParameter is assigned as a member variable of a class like this, try it!

![jobparameter3](./images/5/jobparameter3.png)

You can receive and use JobParameter normally.
This is because the \*\*SimpleJobTasklet bean was created with `@StepScope`.

On the other hand, if this SimpleJobTasklet Bean is created as a general singleton bean, an error like `'jobParameters' cannot be found` occurs as shown below.

![jobparameter4](./images/5/jobparameter4.png)

In other words, it is okay to create a bean through a method or a class, but you can see that the scope of the bean must be Step or Job.

Don't forget that you must create beans with \*\*`@StepScope` and `@JobScope` to use JobParameters.

## 5-4. JobParameter vs System Variable

Looking at the previous story, you may have this question.

- Why do you have to use Job Parameters?
- Can't we use various environment variables or system variables that have been used in Spring Boot?
- If you are using CommandLineRunner, shouldn't you set system variables with `java jar application.jar -D parameter`?

So, let me explain why you need to use Job Parameters.
Let's take a look at the two codes below.

### JobParameter

```java
@Bean
@StepScope
public FlatFileItemReader<Partner> reader(
         @Value("#{jobParameters[pathToFile]}") String pathToFile){
     FlatFileItemReader<Partner> itemReader = new FlatFileItemReader<Partner>();
     itemReader.setLineMapper(lineMapper());
     itemReader.setResource(new ClassPathResource(pathToFile));
     return itemReader;
}
```

### system variables

> The system variables discussed here include application.properties and variables executed with the `-D` option.

```java
@Bean
@ConfigurationProperties(prefix = "my.prefix")
protected class JobProperties {

     String pathToFile;
     ...getters/setters
}

@Autowired
private JobProperties jobProperties;

@Bean
public FlatFileItemReader<Partner> reader() {
     FlatFileItemReader<Partner> itemReader = new FlatFileItemReader<Partner>();
     itemReader.setLineMapper(lineMapper());
     String pathToFile = jobProperties.getPathToFile();
     itemReader.setResource(new ClassPathResource(pathToFile));
     return itemReader;
}
```

There are several differences between the above two methods.

First of all, if you use a system variable, **Spring Batch's Job Parameter related function will be unusable**.
For example, Spring Batch **does not run the same Job twice with the same JobParameter**.
However, if you end up using system variables, this feature doesn't work at all.
Also, meta tables related to parameters automatically managed by Spring Batch are not managed at all.

Second, it is difficult to run a job by any method other than the command line.
If you need to run it, you need to configure Spring Batch to dynamically change the global state (system variable or environment variable) over and over again.
Problems can arise when you want to run multiple jobs at the same time, or when you need to run jobs with test code.

In particular, the fact that the Job Parameter cannot be used is a major drawback.
Not being able to use Job Parameter means that you cannot do **Late Binding** mentioned above.

For example, let's say you have a web server and you want to run a batch on this web server.
If Batch should operate differently depending on the parameters passed from the outside, it is too difficult to solve it as a system variable.
However, if you use the Job Parameter as shown below, you can solve it very easily.

```java
@Slf4j
@RequiredArgsConstructor
@RestController
public class JobLauncherController {

     private final JobLauncher jobLauncher;
     private final job job;

     @GetMapping("/launchjob")
     public String handle(@RequestParam("fileName") String fileName) throws Exception {

         try {
             JobParameters jobParameters = new JobParametersBuilder()
                                     .addString("input.file.name", fileName)
                                     .addLong("time", System.currentTimeMillis())
                                     .toJobParameters();
             jobLauncher.run(job, jobParameters);
         } catch (Exception e) {
             log.info(e.getMessage());
         }

         return "Done";
     }
}
```

If you look at the example, the value received as Request Parameter from Controller is created as Job Parameter.

```java
JobParameters jobParameters = new JobParametersBuilder()
                         .addString("input.file.name", fileName)
                         .addLong("time", System.currentTimeMillis())
                         .toJobParameters();
```

Then, the Job is executed with the created Job Parameter.

```java
jobLauncher.run(job, jobParameters);
```

In other words, you can see that the developer can create job parameters and execute jobs at any timing he wants.
As each Batch component can use Job Parameters, you can easily respond to **even if the changes are severe**.

> Managing batches on a web server is **not recommended**
> The above code is for example only.
> We will introduce later in the series how to manage Spring Batch in a real production environment.

## 5-5. caution

As you can see from the code, using `@Bean` and `@StepScope` together is the same as saying `@Scope (value = "step", proxyMode = TARGET_CLASS)` .

![stepscope](./images/5/stepscope3.png)

This proxyMode can cause problems.
Be sure to read the previously written [Notes on Using @StepScope](http://jojoldu.tistory.com/132) for what kind of problem there is and how to solve it! Please refer to it.

> Same goes for `@JobScope`.

Now, how much did you understand about the scope of Spring Batch through this time?
It is a concept as important as Chunk in Spring Batch, so you must be familiar with it.
Then see you in the next part.
