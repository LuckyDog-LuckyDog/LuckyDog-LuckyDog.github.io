---
title: 写入Excel的两种技术--POI和JXLS
date: 2020-03-14 23:25:31
tags: 
- POI
- JXLS
categories: 
- Java
---

本章通过POI 和 JXLS 各个技术 ,分别应用于项目中

<!--more-->

[TOC]

# 写入Excel的两种技术

目标 , 将前端的数据写入excel , 供管理者决策

![8lbF4e.png](https://s1.ax1x.com/2020/03/14/8lbF4e.png)

## 一 . POI技术

POI 中,使用：XSSF － 提供读写Microsoft Excel OOXML
XSSFWorkbook：工作簿
XSSFSheet：工作表
XSSFRow：行
XSSFCell：单元格
- 创建工作簿的时候, 不需要传入参数(excel不存在的)
- 使用输出流，输出excel
  读取Excel, 每一行读取到了String[] 里面, 多行就是多个String[] , 最终封装到List


### 代码实现
```java
//方式一:利用POI技术将前端数据挨个写入到excel的单元格
        try {

            //获得Excel模板文件绝对路径,separator为了兼容所有//
            String temlateRealPath = request.getSession().getServletContext().getRealPath("template") +
                    File.separator + "report_template.xlsx";
            XSSFWorkbook workbook = new XSSFWorkbook(new FileInputStream(new File(temlateRealPath)));
            XSSFSheet sheet = workbook.getSheetAt(0);

            Map<String, Object> map = reportService.getBusinessReport();
            String reportDate = (String) map.get("reportDate");
            Long todayNewMember = (Long) map.get("todayNewMember");
            Long totalMember = (Long) map.get("totalMember");
            Long thisWeekNewMember = (Long) map.get("thisWeekNewMember");
            Long thisMonthNewMember = (Long) map.get("thisMonthNewMember");
            Long todayOrderNumber = (Long) map.get("todayOrderNumber");
            Long thisWeekOrderNumber = (Long) map.get("thisWeekOrderNumber");
            Long thisMonthOrderNumber = (Long) map.get("thisMonthOrderNumber");
            Long todayVisitsNumber = (Long) map.get("todayVisitsNumber");
            Long thisWeekVisitsNumber = (Long) map.get("thisWeekVisitsNumber");
            Long thisMonthVisitsNumber = (Long) map.get("thisMonthVisitsNumber");
            List<Map> hotSetmealList = (List) map.get("hotSetmeal");
            XSSFRow row = sheet.getRow(2);
            row.getCell(5).setCellValue(reportDate);

            row = sheet.getRow(4);
            row.getCell(5).setCellValue(todayNewMember);
            row.getCell(7).setCellValue(totalMember);

            row = sheet.getRow(5);
            row.getCell(5).setCellValue(thisWeekNewMember);
            row.getCell(7).setCellValue(thisMonthNewMember);

            row = sheet.getRow(7);
            row.getCell(5).setCellValue(todayOrderNumber);
            row.getCell(7).setCellValue(todayVisitsNumber);

            row = sheet.getRow(8);
            row.getCell(5).setCellValue(thisWeekOrderNumber);
            row.getCell(7).setCellValue(thisWeekVisitsNumber);

            row = sheet.getRow(9);
            row.getCell(5).setCellValue(thisMonthOrderNumber);
            row.getCell(7).setCellValue(thisMonthVisitsNumber);

            Integer rowNumber = 12;
            for (Map setmealMap : hotSetmealList) {
                String name = (String) setmealMap.get("name");
                Long setmeal_count = (Long) setmealMap.get("setmeal_count");
                BigDecimal proportion = (BigDecimal) setmealMap.get("proportion");
                String remark = (String) setmealMap.get("remark");
                XSSFRow sheetRow = sheet.getRow(rowNumber);
                sheetRow.getCell(4).setCellValue(name);
                sheetRow.getCell(5).setCellValue(setmeal_count);
                sheetRow.getCell(6).setCellValue(proportion.doubleValue());
                sheetRow.getCell(7).setCellValue(remark);
                rowNumber++;
            }

            ServletOutputStream outputStream = response.getOutputStream();
            response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
            response.setHeader("content-Disposition", "attachment;filename=report.xlsx");
            workbook.write(outputStream);
            outputStream.close();
            workbook.close();
            return null;
        } catch (Exception e) {
            e.printStackTrace();
            return new Result(false, MessageConstant.GET_BUSINESS_REPORT_FAIL);

        }
```

## 二. JXLS

### 代码实现

![8lbl4g.png](https://s1.ax1x.com/2020/03/14/8lbl4g.png)

```java
//方式二:利用JXLS技术将前端数据挨个写入到excel的单元格
try{
//获得Excel模板文件绝对路径
            String temlateRealPath = request.getSession().getServletContext().getRealPath("template") +
                    File.separator + "report_template.xlsx";
            XSSFWorkbook workbook = new XSSFWorkbook(new FileInputStream(new File(temlateRealPath)));
            Map<String, Object> map = reportService.getBusinessReport();
            XLSTransformer xlsTransformer = new XLSTransformer();
            xlsTransformer.transformWorkbook(workbook,map);
      ServletOutputStream outputStream = response.getOutputStream();
            response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
            response.setHeader("content-Disposition", "attachment;filename=report.xlsx");
            workbook.write(outputStream);
            outputStream.close();
            workbook.close();
            return null;
        } catch (Exception e) {
            e.printStackTrace();
            return new Result(false, MessageConstant.GET_BUSINESS_REPORT_FAIL);
        }
```

### (1) . JXLS简介

JXLS是实现了只用几行代码就能创建极其复杂的Excel报表。用特定的标记来创建一个带有规则的.xls模板文件 , 并指定数据放置的位置 , 然后调用JXLS引擎来传递导出的数据放在指定的位置。除了生成Excel报表功能，JXLS还提供了jxls-reader模块，jxls-reader模块会很有用，如果你需要解析一个预定义格式的Excel文件并在其中插入数据的话。

### (2)．JXLS安装

为了使用JXLS引擎，你必须把jxls-core.jar添加到项目的classpath，如果计划使用JXLS来读取.xls文件，那么你必须还要把jxls-reader.jar加入到项目的classpath中。

引入Maven依赖

```xml
<dependency>
    <groupId>net.sf.jxls</groupId>
    <artifactId>jxls-core</artifactId>
    <version>1.0.6</version>
</dependency>
```

### (3)．JXLS用法

#### 1. 属性访问

```java
      Map<String,Object> result = new HashMap<>();
        result.put("reportDate",today);
 XLSTransformer transformer = new XLSTransformer();
transformer.transformXLS(xlsTemplateFileName,result);
```

访问Excel单元格中简单的bean属性：

```java
${reportDate}//多级 , 就 benas.beans1 .beans2 .Filed
```

reportDate属性的值会被映射到Excel单元格中。

#### 2. 多属性遍历

```
<jx:forEach  items="${X}"  var="Y" >   
${Y.FILED}
</jx:forEach>
```

筛选功能

‘select’属性来选择把哪些记录包含在循环中，例如，< 1的数，我们可以使用下面的语句：

```
<jx:forEach items="${X}"  var="Y" select="${Y.Z > 1}">
    ${Y.FILED1}| ${Y.FILED2} | ${Y.FILED3}
</jx:forEach>
```

varStatus 属性

varStatus属性用来定义一个循环状态的名字，在每一次迭代中，循环状态对象会被传递到bean上下文。循环状态对象是LoopStatus类的一个实例，LoopStatus类有一个单一（静态）的'index'属性用来确定当前记录在集合中的索引值（索引值从0开始）。

```
<jx:forEach   items="${X}"  var="Y"  varStatus="status">
      ${status.index}| ${Y.FILED1}| ${Y.FILED2} | ${Y.FILED3}
 </jx:forEach>
```

#### 3. 条件判断

```
<jx:if  test="${X.Filed > 2000.0}">
     ${Y.Filed}
</jx:if>
```

jx:if标签可以基于某些条件来排除某些行或是某些列
如果你把jx:if标签的开始标签和结束标签放在同一行的话，JXLS会根据test的条件来处理或删除包含在标签提内的列；
如果你把jx:if标签的开始标签和结束标签放在不同行的话，JXLS会根据test的条件来处理或删除包含在标签提内的行。

#### 4. 折叠

<jx:outline>标记有一个可选的布尔类型的属性“detail”声明结果初始化的状态-它们应该展开显示还是折叠显示，默认值是false即分组的行会被折叠显示（隐藏）

```
<jx:out>
	 ${Y.Filed}
</jx:out>
```

#### 5. 融合SQL

执行SQL查询，并在Excel文件中显示查询结果，你必须在模板进行转换之前把一个特殊的bean写入bean上下文，这个特殊的bean要实现ReportManager接口。目前这个接口只有一个方法：

```java
public Listexec(String sql) throws SQLException
```

这个方法的参数是一个SQL查询语句，执行结果，返回一个list(list的泛型是bean)

JXLS对这个接口提供了一个默认的实现，叫做ReportManagerImpl，ReportManagerImpl使用owSetDynaClass来把ResultSet对象封装到对象集合中，下面是这个类的用法：

```java
Map beans = newHashMap();
ReportManager rm =new ReportManagerImpl( conn, beans );
beans.put("rm",rm);
InputStream is =new BufferedInputStream(new FileInputStream("reportTemplate.xls"));
XLSTransformertransformer = new XLSTransformer();
HSSFWorkbookresultWorkbook = transformer.transformXLS(is, beans);
```

可以执行任何SQL查询语句通过把它作为参数传递给rm.exec()方法，例如：

```
${rm.exec("SELECT Filed1, Filed2 FROM X")}
```

与 jx:forEach 标签结合起来使用来遍历bean的集合 (ResultSet) 并把它显示在Excel文件中

```xml
<jx:forEachitems="${rm.exec('SELECT y.FILED1, y.FILED2, y.FILED3 FROM Y y')}" var="Y">
 	  ${Y.FILED1}| ${Y.FILED2} | ${Y.FILED3}
</jx:forEach>
```

##### SQL子查询

（子查询）用两个jx:forEach标签，当其中一个标签嵌套在令一个标签里面

```xml
<jx:forEach items="${rm.exec('SELECT d.name, d.id FROM department d')}"var="dep">
     Department:${dep.name}
     Name |Payment | Bonus | Total
   <jx:forEach items="${rm.exec('SELECT name,age, payment, bonus, birthDate FROM employee e where e.depid = ' +dep.id)}" var="employee">
      ${employee.empname}|${employee.payment}|${employee.bonus}|$[B23*(1+C23)]
   </jx:forEach>
</jx:forEach>`
```

##### 语句包含参数

使用外部参数

```java
Map beans = new HashMap();
ReportManager reportManager = new ReportManagerImpl(conn, beans );
beans.put("rm", reportManager);
beans.put("minDate", "1979-01-01");
XLSTransformer transformer = new XLSTransformer();
transformer.transformXLS(templateFileName, beans,destFileName);
```

上面我们把日期“1979-01-01”放到bean的上下文中，以minDate作为key,下面我们使用它来构建一条查询语句：

```xml
<jx:forEachitems="${rm.exec("SELECT d.name depname, e.name empname, age,payment, bonus, birthDate FROM employee e, department d WHERE d.id = e.depidAND birthDate > '1975-01-01' AND birthDate < '" + minDate + "'order by age desc")}" var="employee">
```

可以从上面的语句中了解到如何在SQL查询语句中使用单引号。
