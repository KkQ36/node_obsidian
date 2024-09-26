# Quartz 简介
Quartz 是一个强大的开源定时任务调度框架，用于在 Java 应用程序中执行计划任务。它支持复杂的任务调度功能，能够管理定时任务的执行，提供高灵活性和可扩展性。
类似于 `java.uti.Timer`，它是一个功能强大的开源作业调度库，几乎可以集成到任何 Java 程序中。
包含了很多企业级的功能，例如对 JTA 事务 和 集群的支持，并且可以整合 Spring 使用。
官网：[https://www.quartz-scheduler.org/](https://www.quartz-scheduler.org/)
Maven 坐标：
```xml
<!-- https://mvnrepository.com/artifact/org.quartz-scheduler/quartz -->
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.3.0</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.quartz-scheduler/quartz-jobs -->
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz-jobs</artifactId>
    <version>2.3.0</version>
</dependency>
```
# Quartz 常见 API
## 基本架构
**Job（任务）**，一个具体的任务实现类，我们自己编写的内容，通过实现 Job 接口，然后就可以编写执行逻辑了。
![[Pasted image 20240921110006.png|700]]
**Tigger(触发器)**，定义任务的调度计划，控制任务何时触发。最常用的触发器是 SimpleTrigger 和 CronTrigger。
![[Pasted image 20240921110048.png|700]]
**Scheduler**：调度器，负责管理任务的执行，可以启动、暂停、停止任务。
![[Pasted image 20240921110511.png|650]]
## 核心 API
Quartz 的核心 API 主要集中在任务调度、触发器、作业细节以及调度器管理等方面。下面列举 Quartz 的几个关键核心 API：
- **Job 接口**：所有需要执行的任务都必须实现该接口，定义任务执行逻辑。
- **JobDetail 类**：表示作业的具体细节，如作业的名称、分组、任务类等。`JobDetail` 并不保存作业的状态，仅仅是定义作业的实例。
- **Trigger 接口**：定义任务何时被触发，包含多种类型的触发器，最常见的是 `SimpleTrigger` 和 `CronTrigger`。
- **TriggerBuilder 类**：用于创建触发器的构建器类。
- **Scheduler 接口**：Quartz 的核心调度器接口，负责调度任务，管理作业和触发器的执行。
- **JobKey 类**：用于唯一标识一个 `JobDetail`，通过作业名称和组名来区分。
- **TriggerKey 类**：用于唯一标识一个 `Trigger`，通过触发器名称和组名来区分。
- **JobBuilder 类**：创建 `JobDetail` 对象的构建器类。
- **SchedulerFactory 接口**：用于创建 `Scheduler` 实例的工厂类。
- **JobExecutionContext 类**：提供任务执行的上下文信息，包含任务、触发器、调度器的状态等。
# Quartz 入门案例
## 基本代码
`HelloJob`
```java
public class HelloJob implements Job {  
  
    /**  
     * 当任务触发时，执行此方法  
     * @param jobExecutionContext 任务上下文  
     * @throws JobExecutionException Exception  
     */    @Override  
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {  
        System.out.println("hello quartz，执行时间为：" + jobExecutionContext.getFireTime());  
    }  
  
}
```

`QuartzDemoApplicationTests`
```java
@SpringBootTest  
class QuartzDemoApplicationTests {  
  
    @Test  
    void testJob() {  
        try {  
            // 通过调度器工厂，创建任务调度器  
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();  
  
            // 通过任务构建器，来创建 Job 实例（Job Detail）  
            JobDetail jobDetail = JobBuilder.newJob(HelloJob.class).withIdentity("my_hello_job").build();  
  
            // 通过触发器构建器，来创建触发器对象，并配置每隔三秒触发一次  
            SimpleTrigger trigger = TriggerBuilder.newTrigger().withIdentity("my_hello_trigger").  
                    withSchedule(SimpleScheduleBuilder.simpleSchedule().withIntervalInSeconds(3).repeatForever()).build();  
  
            // 使用 Scheduler 将 Job 和 Trigger 进行关联  
            scheduler.scheduleJob(jobDetail, trigger);  
  
            // 启动任务调度器，开始任务调度  
            scheduler.start();  
  
            // 主线程休眠，等待任务运行  
            Thread.sleep(100000);  
        } catch (SchedulerException | InterruptedException e) {  
            throw new RuntimeException(e);  
        }  
    }  
  
}
```

执行效果展示：
![[Pasted image 20240921112735.png]]
## 思考
### 自定义任务是每次都创建新的实例还是单例模式呢？
我们为上面的 `HelloJob` 添加一个无参构造方法
```java
public HelloJob() {  
    System.out.println("new HelloJob");  
}
```
继续运行程序，观察输出结果
![[Pasted image 20240921112945.png]]
### 如何给定时任务传递参数呢？
通过 `JobDetai` 类的 `JobDataMap` 来实现数据的传递。
我们在上面创建出来 `JobDetail` 的时候，就可以获取其中的 `JobDataMap` 属性。
```java
// 通过任务构建器，来创建 Job 实例（Job Detail）  
JobDetail jobDetail = JobBuilder.newJob(HelloJob.class).withIdentity("my_hello_job").build();  
  
// 获取 JobDataMap 来存储状态信息  
JobDataMap jobDataMap = jobDetail.getJobDataMap();  
jobDataMap.put("name", "ryan");  
jobDataMap.put("age", 18);
```
此时，在调度器的运行过程中，这个 `JobDataMap` 的值就无法变动了。
使用 `JobDataMap`
```java
@Override  
@Override  
public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {  
    JobDataMap jobDataMap = jobExecutionContext.getJobDetail().getJobDataMap();  
    System.out.println("name：" + jobDataMap.get("name") + ", age：" + jobDataMap.get("age"));  
    // 内部的修改不会被保存  
    jobDataMap.put("name", "ryan2");  
    System.out.println("hello quartz，执行时间为：" + jobExecutionContext.getFireTime());  
}
```
而且，每个 Job 使用 `JobDataMap` 相当于使用的是一个副本，对其的修改也不会被保存到下一个 Job 中。
![[Pasted image 20240921154649.png]]
其本质是因为，每次调用方法的时候，使用的 `JobDetailImpl` 对象都是唯一的，每次执行任务会执行 `JobDetailImpl` 的 `clone()` 方法
```java
@Override  
public Object clone() {  
    JobDetailImpl copy;  
    try {  
        copy = (JobDetailImpl) super.clone();  
        if (jobDataMap != null) {  
	        // 对 jobDataMap 进行复制
            copy.jobDataMap = (JobDataMap) jobDataMap.clone();  
        }  
    } catch (CloneNotSupportedException ex) {  
        throw new IncompatibleClassChangeError("Not Cloneable.");  
    }  
  
    return copy;  
}
```
### 能否使用 cron 表达式来指定触发时间呢？
#### 什么是 Cron 表达式？
**Cron 表达式**是一种用于在Quartz中定义任务调度规则的字符串。它由6到7个由空格分隔的字段组成，每个字段对应特定的时间单位（秒、分、小时等），从而形成复杂的调度计划。
表达式的基本结构是这样的：
```java
秒 分 时 日 月 星期 [年（可选）]
```
• **秒**：0-59
• **分**：0-59
• **小时**：0-23
• **日期**：1-31
• **月份**：1-12 或 JAN-DEC
• **星期**：1-7 或 SUN-SAT（1=周日）
• **年（可选）**：1970-2099
#### Cron 表达式常见符号
\*：匹配任意值。
,：列出多个可能的值。
-：指定范围。
/：指定增量值（步长）。
?：表示无意义的字段（通常用于“日期”和“星期”两个字段中的其中一个）。
L：在“日期”字段表示“最后一天”，在“星期”字段表示“最后一周几”。
W：用于“日期”字段，表示最近的工作日。
\#：用于“星期”字段，表示“第几个周几”（例如：2#1表示“第一个星期一”）。
#### Cron 表达式案例
这里直接来看几个案例：
**Cron 表达式**是一种用于在Quartz中定义任务调度规则的字符串。它由6到7个由空格分隔的字段组成，每个字段对应特定的时间单位（秒、分、小时等），从而形成复杂的调度计划。
```java
// 每隔 5 秒执行一次
*/5 * * * * ?

// 每天的凌晨 1:00 执行一次
0 0 1 * * ?

// 每天的 8:00 到 10:00 之间每隔 30 分钟执行一次
0 0/30 8-10 * * ?

// 每个月的 15 号的凌晨 2:30 执行一次
0 30 2 15 * ?

// 每周一至周五的上午 10:15 执行
0 15 10 ? * MON-FRI

// 每月最后一天的 23:59 执行一次
0 59 23 L * ?
```
#### 关于 \* 与 ?：
\* 标识，在一个时间字段中，* 表示该字段可以匹配任意值；而 ? 表示“无特定值”。它用于 **“日期”** 或 **“星期”** 字段，表示在这两个字段中不指定具体的值。
? 存在的意义是，日期和星期可能会存在冲突，例如，如果你想在每个月的 10 号执行任务，而不关心这天是星期几，那么你可以在“星期”字段使用 ?，也就是下面这种方法
```java
0 0 12 10 * ?
```

但如果将 ? 改为 \*，其表达含义就变为了：这表示 **每个月的 10 号**，以及 **每周的每一天** 都会在 12 点触发任务。
```java
0 0 12 10 * *
```

#### 在 Quartz 中使用 Cron 表达式
将前面使用的 `SimpleScheduleBuilder` 变为 `CronScheduleBuilder`：
```java
SimpleScheduleBuilder simpleScheduleBuilder = SimpleScheduleBuilder.simpleSchedule().withIntervalInSeconds(3).repeatForever();
CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule("1,3,5,7,9 * * * * ?");  

// 在触发器中使用 CronScheduleBuilder 而非 SimpleScheduleBuilder
Trigger trigger = TriggerBuilder.newTrigger().withIdentity("my_hello_trigger").  
        withSchedule(cronScheduleBuilder).build();
```
上面的案例中，每分钟的 1 3 5 7 9 秒都会执行一次。
![[Pasted image 20240921162731.png]]
# SpringBoot 整合 Quartz 完成任务持久化与动态定时任务
## Quartz 任务持久化简介
Quartz 有一个 **持久化机制**，也称为 **JobStore**，它将任务调度的相关信息存储在数据库中。
通过这种方式，Quartz 可以在重启后继续执行已计划好的任务，而不会因为应用程序重启、崩溃或服务器宕机等情况导致任务丢失。
这种机制可以用于分布式集群环境，在不同节点上共享同一个任务调度数据。
任务调度信息丢失主要指的是当 Quartz 使用默认的 **内存存储方式（RAMJobStore）** 时，任务的状态和调度信息不会被持久化到数据库。一旦应用程序重启或崩溃，Quartz 中存储在内存里的任务调度信息将全部丢失。这意味着，所有正在运行或计划中的任务将无法恢复，调度会从头开始。
## 入门案例
### 持久化与动态添加任务
使用版本为 SpringBoot 3.3.4，其使用的 Quartz 版本为 2.3.2。
这里，我们直接使用 SpringBoot 来整合 Quartz，只需要在你的 SpringBoot 项目的基础上，添加这一个依赖就可以了。
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```
目前使用的版本已经无需自己再创建表格了。
如果版本不支持自动创建的话可以在 `quartz 解压包/docs/dbTables` 路径中找到需要的 SQL 脚本。（最新版本支持自动创建，该文件已经删除）。
编写自定义任务类，与上面不同的是，这里的 `HelloJob` 需要继承 `QuartzJobBean`
```java
public class HelloJob extends QuartzJobBean {  
  
    public HelloJob() {  
        System.out.println("new HelloJob");  
    }  
  
    @Override  
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {  
        JobDataMap jobDataMap = context.getJobDetail().getJobDataMap();  
        System.out.println("name：" + jobDataMap.get("name") + ", age：" + jobDataMap.get("age"));  
        // 内部的修改不会被保存  
        jobDataMap.put("name", "ryan2");  
        System.out.println("hello quartz，执行时间为：" + context.getFireTime());  
    }  
  
}
```

其父类 `QuartzJobBean` 继承了 `Job` 接口，并且实现了 `execute()` 方法，然后将 `executeInternal()` 方法交给子类去实现。
```java
public abstract class QuartzJobBean implements Job {  
  
    /**  
     * This implementation applies the passed-in job data map as bean property     
     * values, and delegates to {@code executeInternal} afterwards.    
     * @see #executeInternal     
     * */    
    @Override  
    public final void execute(JobExecutionContext context) throws JobExecutionException {  
       try {  
          BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);  
          MutablePropertyValues pvs = new MutablePropertyValues();  
          pvs.addPropertyValues(context.getScheduler().getContext());  
          pvs.addPropertyValues(context.getMergedJobDataMap());  
          bw.setPropertyValues(pvs, true);  
       }  
       catch (SchedulerException ex) {  
          throw new JobExecutionException(ex);  
       }  
       executeInternal(context);  
    }  
  
    /**  
     * Execute the actual job. The job data map will already have been    
     * applied as bean property values by execute. The contract is    
     * exactly the same as for the standard Quartz execute method.     
     * @see #execute     
     * */ 
	protected abstract void executeInternal(JobExecutionContext context) throws JobExecutionException;  
  
}
```

创建 Controller，并注入 Scheduler。
```java
@RestController  
@RequestMapping("quartz")  
public class QuartzController {  
    @Resource  
    private Scheduler scheduler;  
}
```

提供添加任务的接口
```java
@PostMapping("/add")  
public String saveJob(Long bizId, String cronExpr) {  
    JobDetail detail = JobBuilder.newJob(HelloJob.class).withIdentity("my_job_" + bizId).build();  
    JobDataMap jobDataMap = detail.getJobDataMap();  
    jobDataMap.put("biz_id", bizId);  
  
    CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder  
            .cronSchedule(cronExpr)  
            .withMisfireHandlingInstructionDoNothing(); // 错过时，不执行任何操作  
  
    CronTrigger trigger = TriggerBuilder.newTrigger()  
            .withIdentity("my_trigger_" + bizId)  
            .withSchedule(cronScheduleBuilder)  
            .build();  
    try {  
        scheduler.scheduleJob(detail, trigger);  
    } catch (SchedulerException e) {  
        throw new RuntimeException(e);  
    }  
    return "success";  
}
```
我们这个接口来动态的添加一个任务：
![[Pasted image 20240922092645.png]]
执行后就可以看到控制台中的输出了，执行效果与上面配置的完全相同。

> [!Warning] 注意
>  当我们第二次重启的时候，需要将 `quartz.jdbc.initialize-schema` 改为 never，否则再次启动会刷新数据库中的数据，导致持久化失败。=

此时再次重启，也可以看到任务能够继续执行。

### 任务的动态暂停
添加一个接口，能够通过 JobKey 来暂停任务 。
```java
@PutMapping("/pause/{bizId}")  
public String pauseJob(@PathVariable Long bizId) {  
    JobKey jobKey = JobKey.jobKey("my_job_" + bizId);  
    try {  
        scheduler.pauseJob(jobKey);  
    } catch (SchedulerException e) {  
        throw new RuntimeException(e);  
    }  
    return "OK";  
}
```
当其执行完第一个任务的时候，我们调用接口使得任务暂停，注意 JobKey 需要与上面设置的保持一致，可以发现，任务不会再继续执行：
![[Pasted image 20240922094720.png]]
### 暂停定时任务的恢复
```java
@PutMapping("/resume/{bizId}")  
public String resumeJob(@PathVariable Long bizId) {  
    JobKey jobKey = JobKey.jobKey("my_job_" + bizId);  
    try {  
        scheduler.resumeJob(jobKey);  
    } catch (SchedulerException e) {  
        throw new RuntimeException(e);  
    }  
    return "OK";  
}
```
### 删除定时任务
```java
@DeleteMapping  
public String deleteJob(Long bizId) {  
    JobKey jobKey = JobKey.jobKey("my_job_" + bizId);  
    try {  
        scheduler.deleteJob(jobKey);  
    } catch (SchedulerException e) {  
        throw new RuntimeException(e);  
    }  
    return "OK";  
}
```

### 立即执行定时任务
```java
@PutMapping("/run/{bizId}")  
public String runJob(@PathVariable Long bizId) {  
    JobKey jobKey = JobKey.jobKey("my_job_" + bizId);  
    try {  
        scheduler.triggerJob(jobKey);  
    } catch (SchedulerException e) {  
        throw new RuntimeException(e);  
    }  
    return "OK";  
}
```

### 定时任务的更新
```java
@PutMapping("/update/{bizId}")  
public String updateJob(@PathVariable Long bizId) {  
	// 获取到 Trigger Key
    TriggerKey triggerKey = TriggerKey.triggerKey("my_trigger_" + bizId);  
    try {  
        CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule("0-10 * * * * ? *")  
                .withMisfireHandlingInstructionDoNothing();  
        CronTrigger trigger = TriggerBuilder.newTrigger()  
                .withIdentity(triggerKey)  
                .withSchedule(cronScheduleBuilder)  
                .build();  
		// 重新调度任务
        scheduler.rescheduleJob(triggerKey, trigger)  
    } catch (SchedulerException e) {  
        throw new RuntimeException(e);  
    }  
    return "OK";  
}
```

