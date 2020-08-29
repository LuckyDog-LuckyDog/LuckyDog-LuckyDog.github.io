---
title: Freemarker静态化页面技术应用
date: 2020-03-09 17:51:35
tags: [Freemarker,html]
categories: [Java]
---

Freemarker静态化页面主要是用于减少查询数据库的频率 , 高并发访问页面 , 每次都会查询数据 , 对服务器的压力非常大 , 可以降少用户访问数据库 , 可以对相对固定的数据生成静态页面

<!--more-->
# FreeMarker简介
FreeMarker是一款用Java语言编写的模板引擎，用它可以通过模板和要改变的数据来生成输出文本(例如HTML网页，配置文件，源代码等)，作为用来实现网页静态化技术的一种手段。FreeMarker的使用率大大超过其他一些技术。对于系统中频繁使用数据库进行查询但是内容更新很小的应用，都可以用FreeMarker将网页静态化，这样就避免了大量的数据库访问请求，从而提高网站的性能。

# 网页静态化技术与缓存技术的比较

共同点：都可以减小数据库的访问压力。
区别：
(1)缓存技术适用于小规模的数据。以及一些经常变动的数据。
(2)网页静态化技术适用于大规模但是变化不太频繁的数据。

# 网页静态化技术的应用场景

(1)新闻门户网站的文章类型频道一般都用到了网页静态化技术。点击新闻直接会跳到静态化的页面。
(2)电商网站的商品详情页也十分常用，我们在存储商品的时候会生成静态化页面，点击商品详情，会直接跳到生成的商品详情的静态化页面。
(3)网页静态化技术可以结合Nginx这种高性能web服务器来提高并发访问量。

# 生成静态文件操作

```java
public static void main(String[] args) throws Exception{
	//1.创建配置类
	Configuration configuration=new Configuration(Configuration.getVersion());
	//2.设置模板所在的目录 
	configuration.setDirectoryForTemplateLoading(new File("D:\\ftl"));
	//3.设置字符集
	configuration.setDefaultEncoding("utf-8");
	//4.加载模板
	Template template = configuration.getTemplate("test.ftl");
	//5.创建数据模型
	Map map=new HashMap();
	map.put("name", "小劉");
	map.put("message", "www.liuqinghui.top！");
	//6.创建Writer对象
	Writer out =new FileWriter(new File("d:\\test.html"));
	//7.输出
	template.process(map, out);
	//8.关闭Writer对象
	out.close();
}
```

**注意** : 后面在应用时 ,将Configuration对象的创建交由Spring框架来完成，并通过依赖注入方式将字符集和模板所在目录注入进去。

# FreeMaker API

## assign
assign指令用于在页面上定义一个变量
（1）定义简单类型

    <#assign linkman="小劉">
    联系人：${linkman}
（2）定义对象类型
    <#assign info={"mobile":"123456789",'address':'想和妳去看海'} >
    电话：${info.mobile}  地址：${info.address}

## include指令
include指令用于模板文件的嵌套
（1）创建模板文件head.ftl

```html
<h1>我是小劉啊</h1>
```
（2）修改入门案例中的test.ftl，在test.ftl模板文件中使用include指令引入上面的模板文件
```html
<#include "head.ftl"/>
```
## if 指令
if指令用于判断
（1）在模板文件中使用if指令进行判断
```html
<#if success=true>
  你已通过实名认证
<#else>  
  你未通过实名认证
</#if>
```
（2）在java代码中为success变量赋值
```html
map.put("success", true);
```
在freemarker的判断中，可以使用= 也可以使用
if指令可以配合assign success=true使用
```html
<#assign success1=true>
<#if success1=true>
  你已通过实名认证1
<#else>  
  你未通过实名认证1
</#if>
```

## list指令
list指令用于遍历
（1）在模板文件中使用list指令进行遍历
```html
<#list goodsList as goods>
  商品名称： ${goods.name} 价格：${goods.price}<br>
</#list>
```
（2）在java代码中为goodsList赋值
```html
List goodsList=new ArrayList();
```
```html
Map goods1=new HashMap();
goods1.put("name", "苹果");
goods1.put("price", 5.8);

Map goods2=new HashMap();
goods2.put("name", "香蕉");
goods2.put("price", 2.5);

goodsList.add(goods1);
goodsList.add(goods2);
   
map.put("goodsList", goodsList);
```

# Freemaker在开源项目应用

1. 导入freemaker的Maven坐标

```
<dependency>
    <groupId>org.freemarker</groupId>
    <artifactId>freemarker</artifactId>
    <version>2.3.23</version>
</dependency>
```

2. 书写模版(只贴一部分)

```java
    <!-- 页面内容 -->
    <div class="contentBox">
        <div class="list-column1">
            <ul class="list">
                <#list  setmealList as setmeal >
                    <li class="list-item">
                        <a class="link-page" href="mobile_setmeal_detail_${setmeal.id}.html">
                            <img class="img-object f-left" src="http://q6nswofxh.bkt.clouddn.com/${setmeal.img}" alt="">
                            <div class="item-body">
                                <h4 class="ellipsis item-title">${setmeal.name}</h4>
                                <p class="ellipsis-more item-desc">${setmeal.remark}</p>
                                <p class="item-keywords">
                                <span>
    <#if setmeal.sex='0'>
        性别不限
    <#else>
        <#if setmeal.sex='1'>
        男
        <#else>
        女
        </#if>
    </#if>
                                </span>
                                    <span>${setmeal.age}</span>
                                </p>
                            </div>
                        </a>
                    </li>
                </#list>
            </ul>
        </div>
    </div>
</div>
```

3. spring配置Configuration对象( spring-freemaker.xml )

```java
    <bean id="freemarkerConfig"
          class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
        <!--指定模板文件所在目录-->
        <property name="templateLoaderPath" value="/WEB-INF/ftl/" />
        <!--指定字符集-->
        <property name="defaultEncoding" value="UTF-8" />
    </bean>
    <context:property-placeholder location="classpath:freemarker.properties"/>
```

4. 配置静态页面输出路径( freemaker.properties )

```java
out_put_path=D:/Java/Project/Health/healthmoblie_web/src/main/webapp/pages
```

5. 后端对套餐(最大)进行增删改操作之后,就可以实现静态化页面 ,

```java
    @Override
    public void edit(Setmeal setmeal, Integer[] checkGroupIds) {
      //..........此处省略redis操作
        //静态页面改变
        List<Setmeal> setmealList = this.findAll();
        FreemarkUtils.generateMobileSetmealListHtml(setmealList, outPath, freeMarkerConfig);
        //如果取消套餐和检查组的关系 , 那么删除静态页面,否则更新
        List<Integer> groupIdList = setMealDao.findCheckGroupIdById(setmeal.getId());
        if(groupIdList==null || groupIdList.size()==0){
  FreemarkUtils.deleteMobileSetmealDetailByOnlyIdHtml(setmeal.getId(),outPath,freeMarkerConfig);
        }else {
            FreemarkUtils.generateMobileSetmealDetailByOnlyIdHtml(setmeal, setMealDao, outPath, freeMarkerConfig);
        }
    }
    /**
     * 新增套餐
     * @param setmeal 套餐bean
     * @param checkGroupIds 勾选的检查组关系
     */
    @Override
    public void add(Setmeal setmeal, Integer[] checkGroupIds) {
        //增减检查组的信息
        setMealDao.add(setmeal);
        //更新检查组和检查项的关系
        setSetMealAndCheckGroup(setmeal.getId(), checkGroupIds);
        jedisPool.getResource().sadd(RedisConstant.SETMEAL_PIC_DB_RESOURCES, setmeal.getImg());
        //新增套餐后需要重新生成静态页面
        generateMobileStaticHtml();
    }

    /**
     * 生成静态页面
     */
    private void generateMobileStaticHtml() {
        //准备模板文件中所需的数据
        List<Setmeal> setmealList = this.findAll();
        //生成套餐列表静态页面
        FreemarkUtils.generateMobileSetmealListHtml(setmealList, outPath, freeMarkerConfig);
        //生成套餐详情静态页面（多个）
        FreemarkUtils.generateMobileSetmealDetailHtml(setmealList, setMealDao, outPath, freeMarkerConfig);
    }

    /**
     * 通过套餐id查找检查组和检查项是否有id , 没有就可以直接删除
     *
     * @param id
     * @param img
     */
    @Override
    public void deleteById(Integer id, String img) {
     //.......此处省略redies操作
        List<Setmeal> setmealList = this.findAll();
        FreemarkUtils.generateMobileSetmealListHtml(setmealList, outPath, freeMarkerConfig);
        FreemarkUtils.deleteMobileSetmealDetailByOnlyIdHtml(id, outPath, freeMarkerConfig);
    }
```

之后的检查组 , 检查项与套餐有关联都要去更新静态页面 , 根据套餐id进行更新

## 抽取出来的FreemarkUtils工具

```java
package xiaoliu.freemarkutils;

import freemarker.template.Configuration;
import freemarker.template.Template;
import freemarker.template.TemplateException;
import org.springframework.web.servlet.view.freemarker.FreeMarkerConfig;
import xiaoliu.dao.SetMealDao;
import xiaoliu.pojo.Setmeal;

import java.io.*;
import java.util.HashMap;
import java.util.List;

/**
 * author:小刘
 * 日期: 2020/3/9 12:18
 */
public class FreemarkUtils {

    /**生成多个套餐ById静态页面
     * @param setmealList      套餐类列表
     * @param setMealDao
     * @param outPath          html输出路径
     * @param freeMarkerConfig freemark的配置
     */
    public static void generateMobileSetmealDetailHtml(List<Setmeal> setmealList, SetMealDao setMealDao,String outPath, FreeMarkerConfig freeMarkerConfig) {
        HashMap<String, Object> dataMap = new HashMap<>();
        for (Setmeal setmeal : setmealList) {
            Integer setmealId = setmeal.getId();
            Setmeal setMeal = setMealDao.findSetMealById(setmealId);
            dataMap.put("setmeal", setMeal);
            String templateName = "setmeal_detail.ftl";
            String htmlName = "mobile_setmeal_detail_" + setmealId + ".html";
            generateHtml(templateName, htmlName, outPath, freeMarkerConfig, dataMap);
        }
    }
    /**生成一个套餐ById静态页面
     * @param setmeal      前端传过来的套餐
     * @param setMealDao
     * @param outPath          html输出路径
     * @param freeMarkerConfig freemark的配置
     */
    public static void generateMobileSetmealDetailByOnlyIdHtml(Setmeal setmeal, SetMealDao setMealDao,String outPath, FreeMarkerConfig freeMarkerConfig) {
        HashMap<String, Object> dataMap = new HashMap<>();
            Integer setmealId = setmeal.getId();
            Setmeal setMeal = setMealDao.findSetMealById(setmealId);
            dataMap.put("setmeal", setMeal);
            String templateName = "setmeal_detail.ftl";
            String htmlName = "mobile_setmeal_detail_" + setmealId + ".html";
            generateHtml(templateName, htmlName, outPath, freeMarkerConfig, dataMap);
    }
    /**生成一个套餐类静态页面
     * @param setmealList      套餐类列表
     * @param outPath          html输出路径
     * @param freeMarkerConfig freemark的配置
     */
    public static void generateMobileSetmealListHtml(List<Setmeal> setmealList,  String outPath, FreeMarkerConfig freeMarkerConfig) {
        HashMap<String, Object> dataMap = new HashMap<>();
        dataMap.put("setmealList", setmealList);
        String templateName = "setmeal.ftl";
        String htmlName = "mobile_setmeal.html";
        generateHtml(templateName, htmlName, outPath, freeMarkerConfig, dataMap);
    }
    /**
     * 根据套餐id删除页面
     * @param setmealId
     * @param outPath
     * @param freeMarkerConfig
     */
    public static void deleteMobileSetmealDetailByOnlyIdHtml(Integer setmealId,String outPath, FreeMarkerConfig freeMarkerConfig){
        String htmlName = "mobile_setmeal_detail_" + setmealId + ".html";
        File file = new File(outPath+"//"+htmlName);
        if(file!=null  ){
            boolean flag = file.delete();
            if(flag){
                System.out.println("删除成功");
            }else {
                System.out.println("删除失败");
            }
        }
    }
    /**
     * @param templateName     模版名称
     * @param htmlName         根据模板生成html的名称
     * @param outPath          html输出路径
     * @param freeMarkerConfig freemark的配置
     * @param dataMap          map的名称根据模版数据 的名称定义(保持一致)
     */
    public static void generateHtml(String templateName, String htmlName, String outPath, FreeMarkerConfig freeMarkerConfig, HashMap<String, Object> dataMap) {
        BufferedWriter writer = null;
        try {
            Configuration configuration = freeMarkerConfig.getConfiguration();
            Template template = configuration.getTemplate(templateName);
            //生成文件路径
            File file = new File(outPath + "//" + htmlName);
            //字节流---字符流---字符缓冲流
            writer = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file)));
            template.process(dataMap, writer);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TemplateException e) {
            e.printStackTrace();
        } finally {
            try {
                writer.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```



# 运行结果

静态页面出现在我们期望的路径上 , 👏

![898Vq1.png](https://s2.ax1x.com/2020/03/09/898Vq1.png)




