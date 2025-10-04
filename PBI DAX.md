# 常用PBI DAX笔记

## 创建一个手工表
### 维度表
``` DAX
杜邦分析指标 = DataTable("一级指标", STRING,
               "二级指标", STRING,
               "三级指标", STRING,  
               {  
                        {"净资产收益率", "销售净利率", "毛利率"},
                        {"净资产收益率", "销售净利率", "销售费用率"},
                        {"净资产收益率", "销售净利率", "管理费用率"},
                        {"净资产收益率", "销售净利率", "财务费用率"}, 
                        {"净资产收益率", "总资产周转率", "营业收入"},
                        {"净资产收益率", "总资产周转率", "平均资产总额"},
                        {"净资产收益率", "权益乘数", "平均资产总额"},
                        {"净资产收益率", "权益乘数", "平均净资产"}
                }  
           )
```
``` DAX
### 简单的维度表
财务指标 = DATATABLE(
    "财务指标", STRING,
    {
        {"收入"},
        {"利润"}
    }
)
```

## 日期维度表的创建
### 创建日期表，英文
``` DAX
DimDate = 
    VAR startdate = DATE(2024, 1, 1)
    VAR enddate = DATE(2024, 12, 31)
    RETURN
    ADDCOLUMNS(
        CALENDAR(startdate, enddate),
        "Year Number", YEAR([Date]),
        "Year", FORMAT([Date], "yyyy"),
        "Month Number", MONTH([Date]),
        "Month", FORMAT([Date], "mmmm"),
        "Quarter Number", ROUNDUP(MONTH([Date])/3, 0),
        "Quarter", "Q"&ROUNDUP(MONTH([Date])/3, 0),
        "WeekNum", WEEKNUM([Date], 2),
        "WeekDay Number", WEEKDAY([Date], 2),
        "WeekDay", WEEKDAY([Date], 2), 
        "YearMonth Number", YEAR([Date])*100+MONTH([Date]),
        "YearMonth", FORMAT([Date], "yyyy-mm")
    )
```
### 创建日期表，中文
``` DAX
日期表 = 
    VAR startdate = DATE(2020, 1, 1)
    VAR enddate = DATE(2030, 12, 31)
    RETURN
    ADDCOLUMNS(
        CALENDAR(startdate, enddate),
        "年份", YEAR([Date]),
        "季度", "Q" & FORMAT([Date], "Q"),
        "月份", MONTH([Date]),
        "月份名称", FORMAT([Date], "MMMM"),
        "星期几", SWITCH(WEEKDAY([Date], 2),1,"周一",2,"周二",3,"周三",4,"周四",5,"周五",6,"周六",7,"周日"),
        "是否周末", IF(WEEKDAY([Date], 2) > 5, "是", "否")
)
```
### 创建日期表，根据事实表，只创建Date，后续需足迹创建年月周日等字段
``` DAX
日期表=CALENDARAUTO()
```

## 日期计算
### 周环比 
``` DAX
周环比金额 = CALCULATE([Sales Amount], DATEADD('DimDate'[Date], -7, DAY))
```
### 同比环比
``` DAX
环比 = 
VAR CurrentValue = SUM('表名'[数值列]) 
VAR PreviousValue = CALCULATE( SUM('表名'[数值列]), DATEADD('日期表'[日期], -1, MONTH) // 假设按月份比较，-1表示上一个月 ) 
RETURN IF(PreviousValue = 0, 0, (CurrentValue - PreviousValue) / PreviousValue)
同比 = 
VAR CurrentValue = SUM('表名'[数值列]) 
VAR PreviousYearValue = CALCULATE( SUM('表名'[数值列]), SAMEPERIODLASTYEAR('日期表'[日期]) // 自动匹配去年同期 ) 
RETURN IF(PreviousYearValue = 0, 0, (CurrentValue - PreviousYearValue) / PreviousYearValue)
```

### YTD QTD
``` DAX
销售额 YTD = TOTALYTD( SUM('销售表'[销售额]), '日期表'[日期] )
财年销售额 YTD = TOTALYTD( SUM('销售表'[销售额]), '日期表'[日期], , // 无额外筛选时留空 
3 // 年度结束于3月 )

销售额 QTD = TOTALQTD( SUM('销售表'[销售额]), '日期表'[日期] )
```

## 占比，产品分类占比
假定产品维度为：产品类别，产品名称
绝对占比，占比分母为维度下所有成员。
``` DAX
总体占比 = DIVIDE([销售额：合集], CALCULATE([销售额：合集], ALL('产品')))
分类占比 = DIVIDE([销售额：合集], CALCULATE([销售额：合集], ALL('产品'[产品名称])))
```
相对占比，占比分母为筛选器选中的成员。
``` DAX
总体占比 = DIVIDE([销售额：合集], CALCULATE([销售额：合集], ALLSELECTED('产品')))
分类占比 = DIVIDE([销售额：合集], CALCULATE([销售额：合集], ALLSELECTED('产品'[产品名称])))
```

## Filter
### 基本：
``` DAX
华东地区销售 = FILTER(Sales, Sales[Region] = "华东")
```
### 配合聚合函数：
``` DAX
华东总销售额 = SUMX( FILTER(Sales, Sales[Region] = "华东"),  Sales[Amount] )
```
### 与CLACULATE配合
``` DAX
高价产品销售额 = CALCULATE( SUM(Sales[Amount]), FILTER(Sales, Sales[UnitPrice] > 100) )
```
### 多条件：
``` DAX
华东大额销售 = FILTER( Sales, Sales[Region] = "华东" && Sales[Amount] > 1000 )
```

## SWITCH
``` DAX
SWITCH(
    TRUE(),  // 表达式为TRUE，通过后续条件判断匹配
    SELECTEDVALUE('财务指标'[财务指标])="收入", [收入],  // 若选择“收入”，返回[收入]度量值
    SELECTEDVALUE('财务指标'[财务指标])="利润", [利润贡献],  // 若选择“利润”，返回[利润贡献]度量值
    BLANK()  // 无匹配时返回空值
)
```

## 累计占比
``` DAX
累计占⽐% =
VAR vSales = [Sales]
VAR SumALL = SUMX( ALLSELECTED( 'Dim 产品'[产品⼦类别] ) , [Sales] )
VAR RunningSum =
SUMX( FILTER( ALLSELECTED( 'Dim 产品'[产品⼦类别] ) , [Sales] >= vSales ) , [Sales] )
RETURN DIVIDE( RunningSum , SumALL )
```
