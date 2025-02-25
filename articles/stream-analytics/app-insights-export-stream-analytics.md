---
title: Export using Stream Analytics from Azure Application Insights | Microsoft Docs
description: Stream Analytics can continuously transform, filter and route the data you export from Application Insights.
ms.topic: conceptual
ms.date: 01/08/2019
ms.service: stream-analytics
---

# Use Stream Analytics to process exported data from Application Insights

[Azure Stream Analytics](https://azure.microsoft.com/services/stream-analytics/) is the ideal tool for processing data [exported from Application Insights](../azure-monitor/app/export-telemetry.md). Stream Analytics can pull data from a variety of sources. It can transform and filter the data, and then route it to a variety of sinks.

In this example, we'll create an adaptor that takes data from Application Insights using continuous export, renames and processes some of the fields, and pipes it into Power BI.

> [!WARNING]
> There are much better and easier [recommended ways to display Application Insights data in Power BI](../azure-monitor/app/export-power-bi.md). The path illustrated here is just an example to illustrate how to process exported data.

> [!IMPORTANT]
> Continuous export has been deprecated and is only supported for classic Application Insights resources. [Migrate to a workspace-based Application Insights resource](../azure-monitor/app/convert-classic-resource.md) to use [diagnostic settings](../azure-monitor/app/export-telemetry.md#diagnostic-settings-based-export) for exporting telemetry.


![Block diagram for export through Stream Analytics to PowerBI.](./media/app-insights-export-stream-analytics/020.png)

## Create storage in Azure
Continuous export always outputs data to an Azure Storage account, so you need to create the storage first.

1. Create a "classic" storage account in your subscription in the [Azure portal](https://portal.azure.com).
   
   ![Screenshot of the Azure portal, choose New, Data, Storage.](./media/app-insights-export-stream-analytics/030.png)
2. Create a container
   
    ![Screenshot of the new storage, select Containers, click the Containers tile, and then Add.](./media/app-insights-export-stream-analytics/040.png)
3. Copy the storage access key
   
    You'll need it soon to set up the input to the stream analytics service.
   
    ![Screenshot of storage, open Settings, Keys, and take a copy of the Primary Access Key.](./media/app-insights-export-stream-analytics/045.png)

## Start continuous export to Azure storage

[Continuous export](../azure-monitor/app/export-telemetry.md) moves data from Application Insights into Azure storage. 

1. In the Azure portal, browse to the Application Insights resource you created for your application.
   
    ![Screenshot of all resources in the Azure portal, Choose  Application Insights then your application.](./media/app-insights-export-stream-analytics/050.png)
2. Create a continuous export.
   
    ![Screenshot of Application Insights and choose Settings, Continuous Export, then Add.](./media/app-insights-export-stream-analytics/060.png)

    Select the storage account you created earlier:

    ![Screenshot of Continuous export, select with add  then set the export destination.](./media/app-insights-export-stream-analytics/070.png)

    Set the event types you want to see:

    ![Screenshot of add continuous export and choose event types.](./media/app-insights-export-stream-analytics/080.png)

1. Let some data accumulate. Sit back and let people use your application for a while. Telemetry will come in and you'll see statistical charts in [metric explorer](../azure-monitor/essentials/metrics-charts.md) and individual events in [diagnostic search](../azure-monitor/app/diagnostic-search.md).
   
    And also, the data will export to your storage. 
2. Inspect the exported data. In Visual Studio, choose **View / Cloud Explorer**, and open Azure / Storage. (If you don't have this menu option, you need to install the Azure SDK: Open the New Project dialog and open Visual C# / Cloud / Get Microsoft Azure SDK for .NET.)
   
    ![Screenshot showing how to set the event types that you want to see.](./media/app-insights-export-stream-analytics/04-data.png)
   
    Make a note of the common part of the path name, which is derived from the application name and instrumentation key.

The events are written to blob files in JSON format. Each file may contain one or more events. So we'd like to read the event data and filter out the fields we want. There are all kinds of things we could do with the data, but our plan today is to use Stream Analytics to pipe the data to Power BI.

## Create an Azure Stream Analytics instance

From the [Azure portal](https://portal.azure.com/), select the Azure Stream Analytics service, and create a new Stream Analytics job:

![Screenshot that shows the main page for creating Stream Analytics job in the Azure portal.](./media/app-insights-export-stream-analytics/001.png)

![Screenshot that shows the details needed when creating a new Stream Analytics job.](./media/app-insights-export-stream-analytics/002.png)

When the new job is created, select **Go to resource**.

![Screenshot that shows the message received when the new Stream Analytics job deployment is successful.](./media/app-insights-export-stream-analytics/003.png)

### Add a new input

![Screenshot that shows how to add inputs to the Stream Analytics job.](./media/app-insights-export-stream-analytics/004.png)

Set it to take input from your Continuous Export blob:

![Screenshot that shows configuring the Stream Analytics job to take input from a Continuous Export blob.](./media/app-insights-export-stream-analytics/0005.png)

Now you'll need the Primary Access Key from your Storage Account, which you noted earlier. Set this as the Storage Account Key.

### Set path prefix pattern

**Be sure to set the Date Format to YYYY-MM-DD (with dashes).**

The Path Prefix Pattern specifies where Stream Analytics finds the input files in the storage. You need to set it to correspond to how Continuous Export stores the data. Set it like this:

`webapplication27_12345678123412341234123456789abcdef0/PageViews/{date}/{time}`

In this example:

* `webapplication27` is the name of the Application Insights resource **all lower case**.
* `1234...` is the instrumentation key of the Application Insights resource, **omitting dashes**. 
* `PageViews` is the type of data you want to analyze. The available types depend on the filter you set in Continuous Export. Examine the exported data to see the other available types, and see the [export data model](../azure-monitor/app/export-data-model.md).
* `/{date}/{time}` is a pattern written literally.

> [!NOTE]
> Inspect the storage to make sure you get the path right.

## Add new output

Now select your job > **Outputs** > **Add**.

![Screenshot that shows selecting your Stream Analytics job to add a new output.](./media/app-insights-export-stream-analytics/006.png)


![Screenshot of new output, select the new channel, Outputs, Add, then Power BI.](./media/app-insights-export-stream-analytics/010.png)

Provide your **work or school account** to authorize Stream Analytics to access your Power BI resource. Then invent a name for the output, and for the target Power BI dataset and table.

## Set the query

The query governs the translation from input to output.

Use the Test function to check that you get the right output. Give it the sample data that you took from the inputs page.

### Query to display counts of events

Paste this query:

```SQL
SELECT
  flat.ArrayValue.name,
  count(*)
INTO
  [pbi-output]
FROM
  [export-input] A
OUTER APPLY GetElements(A.[event]) as flat
GROUP BY TumblingWindow(minute, 1), flat.ArrayValue.name
```

* export-input is the alias we gave to the stream input
* pbi-output is the output alias we defined
* We use [OUTER APPLY GetElements](/stream-analytics-query/apply-azure-stream-analytics) because the event name is in a nested JSON array. Then the Select picks the event name, together with a count of the number of instances with that name in the time period. The [Group By](/stream-analytics-query/group-by-azure-stream-analytics) clause groups the elements into time periods of one minute.

### Query to display metric values

```SQL
SELECT
  A.context.data.eventtime,
  avg(CASE WHEN flat.arrayvalue.myMetric.value IS NULL THEN 0 ELSE  flat.arrayvalue.myMetric.value END) as myValue
INTO
  [pbi-output]
FROM
  [export-input] A
OUTER APPLY GetElements(A.context.custom.metrics) as flat
GROUP BY TumblingWindow(minute, 1), A.context.data.eventtime
```

* This query drills into the metrics telemetry to get the event time and the metric value. The metric values are inside an array, so we use the OUTER APPLY GetElements pattern to extract the rows. "myMetric" is the name of the metric in this case.

### Query to include values of dimension properties

```SQL
WITH flat AS (
SELECT
  MySource.context.data.eventTime as eventTime,
  InstanceId = MyDimension.ArrayValue.InstanceId.value,
  BusinessUnitId = MyDimension.ArrayValue.BusinessUnitId.value
FROM MySource
OUTER APPLY GetArrayElements(MySource.context.custom.dimensions) MyDimension
)
SELECT
  eventTime,
  InstanceId,
  BusinessUnitId
INTO AIOutput
FROM flat
```

* This query includes values of the dimension properties without depending on a particular dimension being at a fixed index in the dimension array.

## Run the job

You can select a date in the past to start the job from.

![Screenshot of Stream Analyics, select the job and click Query. Paste the sample below.](./media/app-insights-export-stream-analytics/008.png)

Wait until the job is Running.

## See results in Power BI

> [!WARNING]
> There are much better and easier [recommended ways to display Application Insights data in Power BI](../azure-monitor/app/export-power-bi.md). The path illustrated here is just an example to illustrate how to process exported data.


Open Power BI with your work or school account, and select the dataset and table that you defined as the output of the Stream Analytics job.

![Screenshot of Power BI, select your dataset and fields.](./media/app-insights-export-stream-analytics/200.png)

Now you can use this dataset in reports and dashboards in [Power BI](https://powerbi.microsoft.com).

![Screenshot shows a report example made from a dataset in Power BI.](./media/app-insights-export-stream-analytics/210.png)

## No data?

* Check that you [set the date format](#set-path-prefix-pattern) correctly to YYYY-MM-DD (with dashes).

## Next steps
* [Continuous export](../azure-monitor/app/export-telemetry.md)
* [Detailed data model reference for the property types and values.](../azure-monitor/app/export-data-model.md)
* [Application Insights](../azure-monitor/app/app-insights-overview.md)
