---
title: 记录定时器Quartz的使用
date: 2020-09-08 14:33:03
tags:
- 定时器
- spring
categories:
- Java
- Quartz
---



在开发小程序客服功能的时候 , 需要定时去刷新临时素材media_id , 记录定时器的搭建以及开发

<!--more-->

# 快速入门

```java
//引入maven坐标(版本太低,会出现某些方法不兼容导致报错)
      <dependency>
            <groupId>org.quartz-scheduler</groupId>
            <artifactId>quartz</artifactId>
            <version>2.3.0</version>
        </dependency>

package com.example.demo.定时器;

import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.Trigger;
import org.quartz.impl.StdSchedulerFactory;

import static org.quartz.JobBuilder.newJob;
import static org.quartz.SimpleScheduleBuilder.simpleSchedule;
import static org.quartz.TriggerBuilder.newTrigger;

/**
 * author:xiaoLiu
 * Description:定时器触发类
 */
public class QuartzTest {
    public static void main(String[] args) {
        try {
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

            JobDetail job = newJob(test.class)
                    .withIdentity("任务名称", "group1")
                    .build();

            Trigger trigger = newTrigger()
                    .withIdentity("触发器名称", "group1")
                    .startNow()
                    .withSchedule(simpleSchedule()
                            .withIntervalInSeconds(10)
                            .repeatForever())
                    .build();

            scheduler.scheduleJob(job, trigger);

            scheduler.start();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
// 定时器任务类
package com.example.demo.定时器;

import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

/**
 * author:xiaoLiu
 * Description:
 */
public class test implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        System.out.println("我被定时器执行了");
    }
}
//quartz.properties配置文件
org.quartz.scheduler.instanceName = MyScheduler
org.quartz.threadPool.threadCount = 3
org.quartz.jobStore.class = org.quartz.simpl.RAMJobStore
```

启动之后 , 可以看到任务大约每隔10s就会被执行一次

```java
18:42:30.116 [MyScheduler_QuartzSchedulerThread] DEBUG org.quartz.core.QuartzSchedulerThread - batch acquisition of 1 triggers
18:42:30.117 [MyScheduler_Worker-1] DEBUG org.quartz.core.JobRunShell - Calling execute on job group1.任务名称
我被定时器执行了
18:42:40.102 [MyScheduler_QuartzSchedulerThread] DEBUG org.quartz.simpl.PropertySettingJobFactory - Producing instance of Job 'group1.任务名称', class=com.example.demo.定时器.test
18:42:40.102 [MyScheduler_QuartzSchedulerThread] DEBUG org.quartz.core.QuartzSchedulerThread - batch acquisition of 1 triggers
18:42:40.102 [MyScheduler_Worker-2] DEBUG org.quartz.core.JobRunShell - Calling execute on job group1.任务名称
我被定时器执行了
18:42:50.102 [MyScheduler_QuartzSchedulerThread] DEBUG org.quartz.simpl.PropertySettingJobFactory - Producing instance of Job 'group1.任务名称', class=com.example.demo.定时器.test
18:42:50.102 [MyScheduler_QuartzSchedulerThread] DEBUG org.quartz.core.QuartzSchedulerThread - batch acquisition of 1 triggers
18:42:50.102 [MyScheduler_Worker-3] DEBUG org.quartz.core.JobRunShell - Calling execute on job group1.任务名称
我被定时器执行了
```
# 项目上的搭建

## 1. 拓展适配任务工厂类

因为Job对象的实例化过程是在Quartz中进行的，Service是在Spring容器当中的，那么如何将他们关联到一起呢 ? **Quartz提供了JobFactory接口**，让我们可以自定义实现创建Job的逻辑。

```java
public interface JobFactory {
    Job newJob(TriggerFiredBundle bundle, Scheduler scheduler) throws SchedulerException;
}
```

**通过实现JobFactory 接口，在实例化Job以后，在通过ApplicationContext 将Job所需要的属性注入即可在Spring与Quartz集成时 用到的org.springframework.scheduling.quartz.SchedulerFactoryBean这个类**。源码如下，我们只看最关键的地方。

```java
// Get Scheduler instance from SchedulerFactory.
        try {
            this.scheduler = createScheduler(schedulerFactory, this.schedulerName);
            populateSchedulerContext();

            if (!this.jobFactorySet && !(this.scheduler instanceof RemoteScheduler)) {
                // Use AdaptableJobFactory as default for a local Scheduler, unless when
                // explicitly given a null value through the "jobFactory" bean property.
                this.jobFactory = new AdaptableJobFactory();//←←←←重点
            }
            if (this.jobFactory != null) {
                if (this.jobFactory instanceof SchedulerContextAware) {
                    ((SchedulerContextAware) this.jobFactory).setSchedulerContext(this.scheduler.getContext());
                }
                this.scheduler.setJobFactory(this.jobFactory);
            }
        }
```

**我们不指定jobFactory，那么Spring就使用AdaptableJobFactory。这个类的实现**

```java
package org.springframework.scheduling.quartz;

import java.lang.reflect.Method;

import org.quartz.Job;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.spi.JobFactory;
import org.quartz.spi.TriggerFiredBundle;

import org.springframework.util.ReflectionUtils;


public class AdaptableJobFactory implements JobFactory {

   
    public Job newJob(TriggerFiredBundle bundle, Scheduler scheduler) throws SchedulerException {
        return newJob(bundle);
    }

    public Job newJob(TriggerFiredBundle bundle) throws SchedulerException {
        try {
            Object jobObject = createJobInstance(bundle);
            return adaptJob(jobObject);
        }
        catch (Exception ex) {
            throw new SchedulerException("Job instantiation failed", ex);
        }
    }

    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
        // Reflectively adapting to differences between Quartz 1.x and Quartz 2.0...
        Method getJobDetail = bundle.getClass().getMethod("getJobDetail");
        Object jobDetail = ReflectionUtils.invokeMethod(getJobDetail, bundle);
        Method getJobClass = jobDetail.getClass().getMethod("getJobClass");
        Class jobClass = (Class) ReflectionUtils.invokeMethod(getJobClass, jobDetail);
        return jobClass.newInstance();// ←←←←←重点
    }

    protected Job adaptJob(Object jobObject) throws Exception {
        if (jobObject instanceof Job) {
            return (Job) jobObject;
        }
        else if (jobObject instanceof Runnable) {
            return new DelegatingJob((Runnable) jobObject);
        }
        else {
            throw new IllegalArgumentException("Unable to execute job class [" + jobObject.getClass().getName() +
                    "]: only [org.quartz.Job] and [java.lang.Runnable] supported.");
        }
    }

}
```
**这里创建了一个Job实例,那就在这里去给Job的属性进行注入就可以了，写一个类继承它，然后复写这个方法进行对Job的注入。**
```java
package com.mallparking.common.factory;

import org.quartz.spi.TriggerFiredBundle;
import org.springframework.beans.factory.config.AutowireCapableBeanFactory;
import org.springframework.scheduling.quartz.AdaptableJobFactory;
import org.springframework.stereotype.Component;

/**
 * author:xiaoLiu
 * Description:
 */
@Component
public class JobFactory extends AdaptableJobFactory {
    /**
     * AutowireCapableBeanFactory接口是BeanFactory的子类
     * 可以连接和填充那些生命周期不被Spring管理的已存在的bean实例
     */
    private AutowireCapableBeanFactory factory;
    
	//这个对象Spring会帮我们自动把类注入进来
    public JobFactory(AutowireCapableBeanFactory factory) {
        this.factory = factory;
    }

    /**
     * 创建Job实例
     */
    @Override
    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
        // 实例化对象
        Object job = super.createJobInstance(bundle);
        // 进行注入（Spring管理该Bean）
        factory.autowireBean(job);
        //返回对象
        return job;
    }
}
```
**接下来把自定义注入继承的AdaptableJobFactory配置到Spring当中去 , 然后在去修改SchedulerFactoryBean里面的jobFactory**(见3.2.3)

## 2. 配置spring-boot下定时器的配置类

```java
package com.mallparking.config;

import org.quartz.Scheduler;
import org.quartz.ee.servlet.QuartzInitializerListener;
import org.quartz.spi.JobFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.PropertiesFactoryBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.quartz.SchedulerFactoryBean;

import java.io.IOException;
import java.util.Properties;

/**
 * author:xiaoLiu
 * Description: 配置定时器
 */
@Configuration
public class SchedulerConfig {
    @Autowired
    private JobFactory jobFactory;

    /**
     * 调度类FactoyBean
     *
     * @return
     * @throws IOException
     */
    @Bean("sechduerFactory")
    public SchedulerFactoryBean schedulerFactoryBean() throws IOException {
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        //设置调度类quartz属性
        schedulerFactoryBean.setQuartzProperties(quartzProperties());
        //设置jobFactory
        schedulerFactoryBean.setJobFactory(jobFactory);
        return schedulerFactoryBean;
    }

    /**
     * 解析quartz.properties文件，填充属性
     *
     * @return
     * @throws IOException
     */
    @Bean
    public Properties quartzProperties() throws IOException {
        PropertiesFactoryBean propertiesFactoryBean = new PropertiesFactoryBean();
//		propertiesFactoryBean.setLocation(new ClassPathResource("/quartz.properties"));
        propertiesFactoryBean.afterPropertiesSet();
        return propertiesFactoryBean.getObject();
    }

    /**
     * quartz初始化监听器
     *
     * @return
     */
    @Bean
    public QuartzInitializerListener initializerListener() {
        return new QuartzInitializerListener();
    }

    /**
     * 根据调度类工厂bean获取调度
     *
     * @return
     * @throws IOException
     */
    @Bean("scheduler")
    public Scheduler scheduler() throws IOException {
        return schedulerFactoryBean().getScheduler();
    }
}

```
## 3. 实践开发
### 3.1 注解cron表达式开发
需求是更新数据库中微信临时素材media_id

```java

/**
 * author:xiaoLiu
 * Description:
 */
@Component
@EnableScheduling
@EnableAsync
public class Task {
    private static final Logger logger = LoggerFactory.getLogger(Task.class);

    @Autowired
    private SysParamService sysParamService;
    @Autowired
    private RestTemplate restTemplate;
    @Autowired
    private SysParamMapper sysParamMapper;


    @Async
    @Scheduled(cron = "0 0 1 * * ?")  //每天凌晨1点刷新一次
    public void refreshMiniAppMediaId() {
        SysParam sysParam = sysParamService.getValue(SysParamConstant.QUNLIAO_TEMP_MEDIAID);
        String mediaValue = sysParam.getValue();
        if (mediaValue == null) {
            return;
        }
        String path =  ZJYConfig.MEDIA_TEMPPATH;
        downloadMediaFile(path,mediaValue);
        String media_id = uploadMediaFile( path);
        sysParam.setValue(media_id);
        sysParamMapper.updateByPrimaryKey(sysParam);
    }

    private String uploadMediaFile( String targetPath) {
        logger.info("上传临时素材");
        HttpHeaders headers = new HttpHeaders();
        String url = WeChatUtil.getMediIdUrl();
        MediaType type = MediaType.parseMediaType("multipart/form-data");
        headers.setContentType(type);
        MultiValueMap<String, Object> form = new LinkedMultiValueMap<>();
        form.add("file", new FileSystemResource(targetPath));
        Map<String, String> map = restTemplate.postForObject(url, new HttpEntity<>(form, headers), Map.class);
        String media_id = map.get("media_id");
        try {
            Files.delete(Paths.get(targetPath));
        } catch (IOException e) {
            e.printStackTrace();
        }
        return media_id;
    }

    private void downloadMediaFile(String targetPath,String mediaId) {
        String url= WeChatUtil.getTempMediaUrl(mediaId);
        ResponseEntity<byte[]> entity = restTemplate.exchange(url, HttpMethod.GET, new HttpEntity<>(new HttpHeaders()), byte[].class);
        try {
            logger.info("获取临时素材并且放进/tmp/media.jpg");
            File file = new File(targetPath);
            if (file.exists()) {
                file.delete();
            }
            Files.write(Paths.get(targetPath), entity.getBody());
        } catch (Exception e) {
            logger.error("下载临时素材文件时出错:[{}]", e);
        }
    }


}

```
### 3.2 simpleSchedule/cronSchdule开发
需求是设定某个时间,然后将信息发送出去
在快速入门的时候 , 有创建一个核心任务调度器Schduler , 为此我们需要创建一个任务的启动或关闭或删除的类,简称任务处理调度类

#### 3.2.1. 声明Schedule ,注入scheduler对quartz Job进行管理
```java
@Component
public class MallScheduler extends BaseServiceImpl {
    Logger logger = LoggerFactory.getLogger(MallwashScheduler.class);
    @Autowired
    private RemoteControllerService remoteControllerService;
    @Autowired
    @Qualifier("scheduler")
    private Scheduler scheduler;

    public void schedulerJob() throws SchedulerException {
        ApplicationContext annotationContext = SpringUtil.getApplicationContext();
        StdScheduler stdScheduler = (StdScheduler) annotationContext.getBean("mySchedulerFactoryBean");//获得上面创建的bean
        Scheduler myScheduler = stdScheduler;
//        startScheduler1(myScheduler);
        myScheduler.start();
    }

    //删除营业时间定时任务
    public void delTimerJob(String code, String businessTime) {
        if (StringUtils.isNotEmpty(businessTime)) {
            String[] split = businessTime.split(",");
            for (int i = 0; i < split.length; i++) {
                delTimerJob(code, i);
            }
        }

    }

    private void delTimerJob(String code, int i) {
        delStartBusinessTime(code, i);
    }

    private void delStartBusinessTime(String code, int i) {
        try {
            scheduler.deleteJob(JobKey.jobKey(code + "startJob" + i
                    , code + "jobGroup"));
            scheduler.deleteJob(JobKey.jobKey(code + "stopJob" + i
                    , code + "jobGroup"));
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    //新增营业时间定时任务
    public void addTimerJob(String code, String businessTime) {
        if ("all".equals(businessTime)) {
            //发送MQ
            MqDTO changeDTO = new MqDTO.Builder(code).buildSimpleCMD(SysParamConstant.SITE_INOPERATION);
            MqDTO ledDTO = new MqDTO.Builder(code).buildCmd(SysParamConstant.SHOW_LED, "洗车", "");
            remoteControllerService.sendMQ(changeDTO, SysParamConstant.TOPIC_CMD_V1, null);
            remoteControllerService.sendMQ(ledDTO, SysParamConstant.TOPIC_CMD_V1, null);
            return;
        }
        String[] split = businessTime.split(",");
        for (int i = 0; i < split.length; i++) {
            addsingleBusinessTime(split[i], code, i);
        }
    }

    private void addsingleBusinessTime(String singlebusinessTime, String code, int i) {
        String[] time = singlebusinessTime.split("_");
        String startTime = time[0];
        String stopTime = time[1];
        addStartBusinessTime(startTime, code, i);
        addStopBusinessTime(stopTime, code, i);
    }

    private void addStartBusinessTime(String startTime, String code, int i) {
        JobDetail jobDetail = JobBuilder.newJob(StartBusinessTask.class).withIdentity(code + "startJob" + i, code + "jobGroup").build();
        jobDetail.getJobDataMap().put("code", code);
        String[] time = startTime.split(":");
        int hour = Integer.parseInt(time[0]);
        int min = Integer.parseInt(time[1]);
        String cron = "0 " + min + " " + hour + " * * ?";//0 0 22 * * ?
        logger.info(code + "startCron表达式:{}", cron);
        CronScheduleBuilder cronScheduleBuilder;
        try {
            cronScheduleBuilder = CronScheduleBuilder.cronSchedule(cron);
        } catch (Exception e) {
            logger.error("[定时任务表达式错误]:", e);
            return;
        }
        CronTrigger trigger = TriggerBuilder.newTrigger().withIdentity(code + "startTrigger" + i, code + "triggerGroup")
                .withSchedule(cronScheduleBuilder).build();
        try {
            scheduler.scheduleJob(jobDetail, trigger);
            scheduler.start();// 触发器并不会立刻触发
        } catch (SchedulerException e) {
            logger.error("[定时任务初始化错误]:", e);
        }
    }

    private void addStopBusinessTime(String stopTime, String code, int i) {
        JobDetail jobDetail = JobBuilder.newJob(StopBusinessTask.class).withIdentity(code + "stopJob" + i, code + "jobGroup").build();
        jobDetail.getJobDataMap().put("code", code);
        String[] time = stopTime.split(":");
        int hour = Integer.parseInt(time[0]);
        int min = Integer.parseInt(time[1].equals("00") ? "0" : time[1]);
        String cron = "0 " + min + " " + hour + " * * ?";//0 0 22 * * ?
        logger.info(code + "stopCron表达式:{}", cron);
        CronScheduleBuilder cronScheduleBuilder = null;
        cronScheduleBuilder = CronScheduleBuilder.cronSchedule(cron);
        CronTrigger trigger = TriggerBuilder.newTrigger().withIdentity(code + "stopTrigger" + i, code + "triggerGroup")
                .withSchedule(cronScheduleBuilder).build();
        try {
            scheduler.scheduleJob(jobDetail, trigger);
            scheduler.start();// 触发器并不会立刻触发
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    /**
     * 设置时间定时推送
     * @param param
     */
    public void addPushTime(EWxMsgTask2Param param) {
        logger.info("任务实体{}",param);
        String startTime = param.getStartTime();
        JobDetail jobDetail = JobBuilder.newJob(StartPushTask.class).withIdentity(startTime + "pushJob", startTime + "jobGroup").build();
        jobDetail.getJobDataMap().put("ids", param.getIds());
        jobDetail.getJobDataMap().put("quaId", param.getQuaId());
        jobDetail.getJobDataMap().put("type",param.getType() );
        jobDetail.getJobDataMap().put("sendType", param.getSendType());
        jobDetail.getJobDataMap().put("quartz", param.getQuartz());
        Date date = null;
        if(startTime!=null){
            try {
                SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                date = sdf.parse(startTime);
            } catch (ParseException e) {
                logger.info("开始时间解析异常");
                e.printStackTrace();
            }
        }else {
            return;
        }

        logger.info("任务开始的时间[{}]",date);
        if (date != null) {
            SimpleScheduleBuilder simpleScheduleBuilder = SimpleScheduleBuilder.simpleSchedule();
            //时间到了就执行一次
            SimpleTrigger trigger = TriggerBuilder.newTrigger().withIdentity(startTime + "pushTrigger", startTime + "triggerGroup").startAt(date)
                    .withSchedule(simpleScheduleBuilder).build();
            try {
                scheduler.scheduleJob(jobDetail, trigger);
                scheduler.start();
            } catch (SchedulerException e) {
                e.printStackTrace();
            }
        }
    }
}

```

#### 3.2.2. 创建任务类继承Job

```java
@Component
public class StartPushTask implements Job {
    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {

        String ids =(String)jobExecutionContext.getJobDetail().getJobDataMap().get("ids");
        Integer type =(Integer)jobExecutionContext.getJobDetail().getJobDataMap().get("type");
        String quaId =(String) jobExecutionContext.getJobDetail().getJobDataMap().get("quaId");
        Integer sendType =(Integer) jobExecutionContext.getJobDetail().getJobDataMap().get("sendType");
     
        EWxMsgTask2Param eWxMsgTask2Param = new EWxMsgTask2Param();
        eWxMsgTask2Param.setIds(ids);
        eWxMsgTask2Param.setType(type);
        eWxMsgTask2Param.setSendType(sendType);

        log.info("开始执行定时任务[{}]",eWxMsgTask2Param);
        try {
            wxSendMsgTaskService.addSendMsgTask2(eWxMsgTask2Param);//时间到推送消息
            WXQuartEntity wxQuartEntity = wxQuartzRespority.findOne(quaId);
            wxQuartEntity.setSendFlag(1);
            log.info("修改数据库的发送类型[{}]",wxQuartEntity);
            wxQuartzRespority.save(wxQuartEntity);
        } catch (Exception e) {
            e.printStackTrace();
            log.error("定时任务错误{}",e.getMessage());
        }
    }
}

//其他类中addSendMsgTask2方法
@Override
    public BaseResponse addSendMsgTask2(EWxMsgTask2Param param) {
        //是定时任务的推送
        if (param.getQuartz() != null && param.getQuartz() == 0) {
            log.info("开始执行定时任务的前置工作");
            WXQuartEntity wxQuartEntity = new WXQuartEntity();
            wxQuartEntity.setSendFlag(0);
            wxQuartEntity.setStartTime(param.getStartTime());
            wxQuartEntity.setUser_ids(param.getIds());
            wxQuartEntity.setType(param.getType());
            WXQuartEntity save = wxQuartzRespority.save(wxQuartEntity);
            param.setQuaId(save.getId());
            mallScheduler.addPushTime(param);
            log.info("将任务数据保存到数据库[{}]", save);
            return BaseResponse.ok("已通知定时任务执行{}", param);
        } }
```

#### 3.2.3. spring-boot启动的时候,需要将任务加载进定时器

```java
/**
 * 启动的时候将任务添加
 */
@Component
public class QuartzApplicationRunner2 implements ApplicationRunner {
    @Autowired
    private MallwashScheduler mallwashScheduler;
    @Autowired
    private WXQuartzRespority wxQuartzRespority;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        Logger log = LoggerFactory.getLogger(MallwashScheduler.class);
        List<WXQuartEntity> wxQuartEntityList = wxQuartzRespority.findBySendFlag(0);
        if(wxQuartEntityList!=null&&wxQuartEntityList.size()>0){
            wxQuartEntityList.stream().forEach(e->{
                log.info("启动开始将未发送加入定时任务数据[{}]",e);
                EWxMsgTask2Param eWxMsgTask2Param = new EWxMsgTask2Param();
                String startTime = e.getStartTime();
                eWxMsgTask2Param.setStartTime(startTime);
                eWxMsgTask2Param.setIds(e.getUser_ids());
                eWxMsgTask2Param.setQuaId(e.getId());
                eWxMsgTask2Param.setType(e.getType());
                eWxMsgTask2Param.setSendType(1);
                mallwashScheduler.addPushTime(eWxMsgTask2Param);// 这里将实体传给上述的addPushTime,通过.start进行执行
                log.info("已加入定时任务实体[{}]",eWxMsgTask2Param);
            });
        }
    }
}
```

#### 3.2.4. 设置spring-boot启动监听

任务启动的时候,将多个任务一起执行

```java
/**
 * @program: 
 * @description:多任务一起执行
 * @author: xiaoLiu
 **/
@Configuration
public class ApplicationStartQuartzJobListener implements ApplicationListener<ContextRefreshedEvent> {

    @Autowired
    MallScheduler mallScheduler;

    @Autowired
    JobFactory jobFactory;


    @Override
    public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {
        try {
            mallScheduler.schedulerJob();
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    @Bean(name = "mySchedulerFactoryBean")
    public SchedulerFactoryBean mySchedulerFactory() {
        SchedulerFactoryBean bean = new SchedulerFactoryBean();
        bean.setOverwriteExistingJobs(true);
        bean.setStartupDelay(1);
        bean.setJobFactory(jobFactory);
        return bean;
    }

}

```
# 参考资料

1. https://www.cnblogs.com/daxin/p/3608320.html
2. https://www.w3cschool.cn/quartz_doc/quartz_doc-2put2clm.html
3. https://blog.csdn.net/lyg_come_on/article/details/78223344