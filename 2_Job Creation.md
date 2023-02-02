# 2. Execute Batch Job

In this time, we will study the overall contents while creating & executing a simple Spring Batch Job.

> All the code I worked on is on [Github](https://github.com/jojoldu/spring-batch-in-action), so you can refer to it.

## 2-1. Creating a Spring Batch project

The basic project development environment is as follows.

- IntelliJ IDEA 2018.2
- Spring Boot 2.0.4
- Java 8
  \*gradle

> I use lombok features a lot.
> It is recommended to install the lombok plugin according to your IDE :)
> [Intellij IDEA](http://blog.woniper.net/229), [Eclipse](http://countryxide.tistory.com/16)

Based on this, let's start creating a project.

> I run IntelliJ Ultimate (paid) version, but Eclipse doesn't seem to have a very different screen configuration.

First select Spring Initializr (Spring Boot) which represents your Spring Boot project.

![project1](./images/2/project1.png)

Select your own Group, Artifact and select Gradle project.
Then, on the screen to select Spring dependencies, select as follows.

![project1](./images/2/project2.png)

> If your project only uses JPA, you do not need to select JDBC.
> Or **If you don't use JPA, you don't have to select JPA**.

The build.gradle will look like this:

```groovy
buildscript {
     ext {
         springBootVersion = '2.0.4.RELEASE'
     }
     repositories {
         mavenCentral()
     }
     dependencies {
         classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
     }
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

group = 'com.jojoldu.spring'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = 1.8

repositories {
     mavenCentral()
}


dependencies {
     compile('org.springframework.boot:spring-boot-starter-batch')
     compile('org.springframework.boot:spring-boot-starter-data-jpa')
     compile('org.springframework.boot:spring-boot-starter-jdbc')
     runtime('com.h2database:h2')
     runtime('mysql:mysql-connector-java')
     compileOnly('org.projectlombok:lombok')
     testCompile('org.springframework.boot:spring-boot-starter-test')
     testCompile('org.springframework.batch:spring-batch-test')
}

```

And if you open `BatchApplication.java` in the package, you will see the `main` method as shown below.

![3](./images/2/project3.png)

Typical Spring Boot code, right?
Now let's create a simple Spring Batch Job.

## 2-2. Creating a Simple Job

Before creating a Batch Job, add **Spring Batch feature enable** annotation (`` @EnableBatchProcessing``) to  `BatchApplication.java``` as follows.

![simplejob1](./images/2/simplejob1.png)

By declaring this annotation, you can use various features of Spring Batch.
If you do not declare it, you cannot use the Spring Batch function, so you must declare it **required**.

When the configuration is complete, create the `job` package under the package, and create the ``SimpleJobConfiguration.java` file.

![simpleJob2](./images/2/simplejob2.png)

Write a simple Spring Batch code named `simpleJob` in the created Java file.

```java

@Slf4j // lombok annotation for using log
@RequiredArgsConstructor // lombok annotation for constructor DI
@Configuration
public class SimpleJobConfiguration {
     private final JobBuilderFactory jobBuilderFactory; // get constructor DI
     private final StepBuilderFactory stepBuilderFactory; // get constructor DI

     @Bean
     public Job simpleJob() {
         return jobBuilderFactory.get("simpleJob")
                 .start(simpleStep1())
                 .build();
     }

     @Bean
     public Step simpleStep1() {
         return stepBuilderFactory.get("simpleStep1")
                 .tasklet((contribution, chunkContext) -> {
                     log.info(">>>>> This is Step1");
                     return RepeatStatus. FINISHED;
                 })
                 .build();
     }
}
```

- `@Configuration`
  - All Jobs in Spring Batch are registered with `@Configuration`.
- `jobBuilderFactory.get("simpleJob")`
  - Create a Batch Job named `simpleJob`.
  - The job name is not specified separately, but is specified through the Builder.
- `stepBuilderFactory.get("simpleStep1")`
  - Create a Batch Step named `simpleStep1`.
  - As with `jobBuilderFactory.get("simpleJob")`, specify the name through the Builder.
- `.tasklet((contribution, chunkContext))`
  - Specifies the functions to be performed within the Step.
  - Tasklet is used when declaring custom functions to be executed singly within a **Step**.
  - Here, `log.info(">>>>> This is Step1")` is output when batch is executed.

If you look at the **simpleJob code that creates a Batch Job, you can see that it has simpleStep1**.
In Spring Batch, **Job refers to one batch work unit**.
Within a Job, there are several Steps as shown below, and within a Step there are Tasklet or Reader & Processor & Writer bundles.

![jobstep](./images/2/jobstep.png)

It is easy to understand that there are several steps in a job, but the unit of a step may seem ambiguous.

**One Tasklet and one set of Reader & Processor & Writer are at the same level**.
So **You must keep in mind that you cannot make it by completing the Reader & Processor and finishing with Tasklet**.

> In a way, you can see Tasklet as a role similar to `@Component` and `@Bean` of Spring MVC.
> It doesn't have a clear role, but you can think of it as a unit for the custom functionality specified by the developer.

Now, let's run this simple Spring Batch application.
Batch is executed when the `main` method of `BatchApplication.java`, which was created first, is executed.

![simpleJob5](./images/2/simpleJob5.png)

If you run it, you can see that `log.info(">>>>> This is Step1")` was successfully executed and the log was taken as shown below.

![simpleJob6](./images/2/simpleJob6.png)

I wrote my first Spring Batch program!
I did a very simple exercise.
This stuff is so simple.
Let's go back to something a little more difficult.

## 2-3. Running Spring Batch in MySQL environment

In the previous process, Spring Batch was performed very simply.
**Spring Batch only requires writing application code**! You may think.
In reality it is not.
Spring Batch requires metadata tables.

> You can think of meta data as **data that describes data**.
> [Namu Wiki link](https://namu.wiki/w/%EB%A9%94%ED%83%80%EB%8D%B0%EC%9D% B4%ED%84%B0).

Spring Batch's metadata contains the following contents.

- What are the previously executed Jobs
- Which Batch Parameters have recently failed and which Jobs have succeeded
- If you run it again, where do you start?
- What kind of Steps were in which Job, and which Steps succeeded and which Steps failed among the Steps

Metadata for operating Batch applications is divided into several tables.
The meta table structure looks like this:

![domain1](./images/2/domain1.png)

( Source: [metaDataSchema](https://docs.spring.io/spring-batch/3.0.x/reference/html/metaDataSchema.html) )

Spring Batch works only with these tables.
Basically, when using H2 DB, the table is automatically created when Boot is executed, but when using DB such as MySQL or Oracle, the developer must create it himself.

Then you might be wondering about the schema of these tables.
The corresponding schema already exists in Spring Batch, just copy it and `create table`.

If you run `schema-` as a file search in your IDE, you can see that each meta table schema exists according to the DBMS.

![schema](./images/2/schema.png)

Then, let's run Spring Batch using MySQL.

### 2-3-1. Connect to MySQL

Let's install MySQL on your PC and add configuration so that Spring Batch uses MySQL.

Add Datasource settings to `src/main/resources/application.yml` of the project as shown below.

```yaml
spring:
  profiles:
    active: local
---
spring:
  profiles: local
  datasource:
    hikari:
      jdbc-url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
      username: sa
      password:
      driver-class-name: org.h2.Driver
---
spring:
  profiles: mysql
  datasource:
    hikari:
      jdbc-url: jdbc:mysql://localhost:3306/spring_batch
      username: jojoldu
      password: jojoldu1
      driver-class-name: com.mysql.cj.jdbc.Driver
```

- `spring.datasource.hikari`
  - The default DataSource of Spring Boot is Hikari (Hikari CP).
- `jdbc-url`
  - Specifies the MySQL address to connect to.
  - I access the database called `spring_batch` by connecting to port 3306 of Localhost.
- ` username``,  `password```
  - I set the installed MySQL account as `jojoldu` and the password as `jojoldu1`.
- `driver-class-name`
  - JDBC driver for MySQL.
  - JDBC driver must be specified for each DBMS.

In the above configuration, each `spring.profiles` means that if the profile is local, H2 is used, and if the profile is set to mysql, MySQL is used\*\*.

Once the setup is complete, let's run it.

### 2-3-2. Running in MySQL environment

I haven't created meta tables in MySQL yet.
Then the batch should fail as described above, right?

Set Spring Batch's profile to mysql in IntelliJ and run it.

Let's leave the existing execution environment as it is for use when running H2, and create a new execution environment.
Click the Execution Environment button at the top.

![mysql1](./images/2/mysql1.png)

Copy the default created execution environment as shown in the image below.

![mysql2](./images/2/mysql2.png)

Change the copied execution environment name and active profiles to mysql.

![mysql3](./images/2/mysql3.png)

Execute with the newly created mysql environment.

![mysql4](./images/2/mysql4.png)

The project's profile runs fine with mysql.

![mysql5](./images/2/mysql5.png)

Scroll down the console a bit more!

![mysql6](./images/2/mysql6.png)

You can see that the batch failed with the error that the meta table data `BATCH_JOB_INSTANCE` does not exist.

Now, let's create a metadata table so that there are no errors.
As mentioned above, search for `schema-mysql.sql` file.

![mysql7](./images/2/mysql7.png)

Copy all the schemas in this file and run it on your local MySQL.
And check if it was created well.

![mysql8](./images/2/mysql8.png)

> I am using JetBrains' [DataGrip](https://www.jetbrains.com/datagrip/) as my DB Client.
> It supports the same UX and shortcuts as IntelliJ, so you can use it comfortably without learning how to use it separately.
> It's free for a month, so I recommend you to try it once :)

Now let's run Spring Batch again with mysql profile.

![mysql9](./images/2/mysql9.png)

Wow!
The batch ran fine on MySQL as well!
Now, let's see what kind of information is contained in this meta table one by one.
