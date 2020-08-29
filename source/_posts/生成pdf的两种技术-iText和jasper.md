---
title: 生成pdf的两种技术--iText和jasper
date: 2020-03-16 18:13:40
tags: [iText,Jasper]
categories: [Java]
---

PDF生成技术的本章学习记录了两种 , 一种是IText , 一种是jasperReport , 推荐使用jasperReport , 因为通过jasper studio可以快速生产格式

<!--more-->

## 项目整合两种技术的效果
### iText整合

![8Y9SRs.png](https://s1.ax1x.com/2020/03/16/8Y9SRs.png)

### Jasper整合

![8Y9AdU.png](https://s1.ax1x.com/2020/03/16/8Y9AdU.png)

## 两种技术的应用
### IText
 通过iText不仅可以生成PDF或rtf的文档，而且可以将XML、Html文件转化为PDF文件

```xml
<!--坐标导入-->
<!--导入Itext报表-->
<dependency>
  <groupId>com.lowagie</groupId>
  <artifactId>itext</artifactId>
  <version>2.1.7</version>
</dependency>
<!-- 导入iText报表，支持中文 -->
<!--<dependency 不兼容原因见图片分析>
  <groupId>com.itextpdf</groupId>
  <artifactId>itext-asian</artifactId>
  <version>5.2.0</version>
</dependency>-->
<dependency>
    <groupId>com.lowagie</groupId>
    <artifactId>itextasian</artifactId>
    <version>1.0</version>
</dependency>
```

上述异常分析

![88UZh4.png](https://s1.ax1x.com/2020/03/15/88UZh4.png)

整合项目

```java
try {
            Map map = orderService.findById(id);
            Integer setmealId = (Integer)map.get("setmealId");
            String membername = (String) map.get("member");
            Setmeal setmeal = setMealService.findById(setmealId);
            response.setContentType("application/pdf");
            String filename = membername+"的预约信息.pdf";
            filename = URLEncoder.encode(filename, "utf-8");//chrome浏览器解码
            // 设置以附件的形式导出
            response.setHeader("Content-Disposition",
                    "attachment;filename=" + filename);

            Document document = new Document();
            PdfWriter pdfWriter = PdfWriter.getInstance(document, response.getOutputStream());
            document.open();
            //创建字体
            BaseFont cn = BaseFont.createFont("STSongStd-Light", "UniGB-UCS2-H", false);
            Font font = new Font(cn, 10, Font.NORMAL, Color.gray);
            // 写PDF数据
            // 输出订单和套餐信息
            // 体检人
            document.add(new Paragraph("体检人："+(String)map.get("member"), font));
            // 体检套餐
            document.add(new Paragraph("体检套餐："+(String)map.get("setmeal"), font));
            // 体检日期
            document.add(new Paragraph("体检日期："+(String) map.get("orderDate").toString(), font));
            // 预约类型
            document.add(new Paragraph("预约类型："+(String)map.get("orderType"), font));
            // 向document 生成pdf表格
            Table table = new Table(3);//创建3列的表格
            table.setWidth(80); // 宽度
            table.setBorder(1); // 边框
            table.getDefaultCell().setHorizontalAlignment(Element.ALIGN_CENTER); //水平对齐方式
            table.getDefaultCell().setVerticalAlignment(Element.ALIGN_TOP); // 垂直对齐方式
            /*设置表格属性*/
            table.setBorderColor(new Color(128, 128, 128)); //将边框的颜色设置为灰色
            table.setPadding(5);//设置表格与字体间的间距
//table.setSpacing(5);//设置表格上下的间距
            table.setAlignment(Element.ALIGN_CENTER);//设置字体显示居中样式

            // 写表头
            table.addCell(buildCell("项目名称", font));
            table.addCell(buildCell("项目内容", font));
            table.addCell(buildCell("项目解读", font));
            // 写数据
            for (CheckGroup checkGroup : setmeal.getCheckGroups()) {
                table.addCell(buildCell(checkGroup.getName(), font));
                // 组织检查项集合
                StringBuffer checkItems = new StringBuffer();
                for (CheckItem checkItem : checkGroup.getCheckItems()) {
                    checkItems.append(checkItem.getName()+"  ");
                }
                table.addCell(buildCell(checkItems.toString(), font));
                table.addCell(buildCell(checkGroup.getRemark(), font));
            }
            // 将表格加入文档
            document.add(table);
            document.close();
            return null;
        } catch (Exception e) {
            return new Result(false, MessageConstant.GET_BUSINESS_REPORT_FAIL,null);
        }

    }
```



### JasperReport

一般情况下，JasperReports会结合Jaspersoft Studio(模板设计器)使用导出PDF报表。方便 , 快捷

引入坐标

```xml
<jasperreports>6.8.0</jasperreports>
<dependency>
    <groupId>net.sf.jasperreports</groupId>
    <artifactId>jasperreports</artifactId>
    <version>${jasperreports}</version>
</dependency>
```

原理

![88d66s.png](https://s1.ax1x.com/2020/03/15/88d66s.png)

由于jasper同样不支持中文 , 所以需要引入三方配置来进行解析中文 , 否则无法显示中文 , 一定要注意

![8J7j56.png](https://s1.ax1x.com/2020/03/16/8J7j56.png)

注意注意 : 这里涉及的**所有中文**和**模板公式**都要设置华文宋体 , 不然中文不出来

![8Y9tWd.png](https://s1.ax1x.com/2020/03/16/8Y9tWd.png)

整合项目

```java
         Map map = orderService.findById(id);
            Map<String, Object> result = new HashMap<>();

            Integer setmealId = (Integer) map.get("setmealId");
            result.put("member",(String) map.get("member"));
            result.put("setmeal",(String)map.get("setmeal"));
            result.put("orderDate",map.get("orderDate").toString());
            result.put("orderType",(String) map.get("orderType"));
            result.put("company","晚风的岁月");

            java.util.List<Object> list = new ArrayList<>();

            Setmeal setmeal = setMealService.findById(setmealId);
            for (CheckGroup checkGroup : setmeal.getCheckGroups()) {
                // 组织检查项集合
                StringBuffer checkItemName = new StringBuffer();
                for (CheckItem checkItem : checkGroup.getCheckItems()) {
                    checkItemName.append(checkItem.getName()+"  ");
                }

                HashMap<String, Object> hashMap = new HashMap<>();
                hashMap.put("checkGroupName",checkGroup.getName());
                hashMap.put("itemName",checkItemName.toString());
                hashMap.put("checkGroupRemark",checkGroup.getRemark());
                list.add(hashMap);
            }


            //动态获取模板文件绝对磁盘路径
            String jrxmlPath =
                    request.getSession().getServletContext().getRealPath("template") + File.separator + "orderInfo.jrxml";
            String jasperPath =
                    request.getSession().getServletContext().getRealPath("template") + File.separator + "orderInfo.jasper";
                //编译模版
            JasperCompileManager.compileReportToFile(jrxmlPath, jasperPath);
            //填充数据---使用JavaBean数据源方式填充
            JasperPrint jasperPrint = JasperFillManager.fillReport(jasperPath,result, new JRBeanCollectionDataSource(list));
            response.setContentType("application/pdf");
            String filename = (String) map.get("member")+"的预约信息.pdf";
            filename = URLEncoder.encode(filename, "utf-8");//chrome浏览器解码
            // 设置以附件的形式导出
            response.setHeader("Content-Disposition", "attachment;filename=" + filename);
            ServletOutputStream out = response.getOutputStream();
            JasperExportManager.exportReportToPdfStream(jasperPrint,out);
```





















































