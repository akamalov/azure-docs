---
title: Fault tolerance of copy activity in Azure Data Factory | Microsoft Docs
description: 'Learn about how to add fault tolerance to copy activity in Azure Data Factory by skipping the incompatible rows.'
services: data-factory
documentationcenter: ''
author: linda33wj
manager: jhubbard
editor: spelluru

ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 09/05/2017
ms.author: jingwang

---
#  Fault tolerance of copy activity in Azure Data Factory
> [!div class="op_single_selector" title1="Select the version of Data Factory service you are using:"]
> * [Version 1 - GA](v1/data-factory-copy-activity-fault-tolerance.md)
> * [Version 2 - Preview](copy-activity-fault-tolerance.md)

The copy activity in Azure Data Factory offers you two ways to handle incompatible rows when copying data between source and sink data stores:

- You can abort and fail the copy activity when incompatible data is encountered (default behavior).
- You can continue to copy all of the data by adding fault tolerance and skipping incompatible data rows. In addition, you can log the incompatible rows in Azure Blob storage. You can then examine the log to learn the cause for the failure, fix the data on the data source, and retry the copy activity.

> [!NOTE]
> This article applies to version 2 of Data Factory, which is currently in preview. If you are using version 1 of the Data Factory service, which is generally available (GA), see [copy activity fault tolerance in V1](v1/data-factory-copy-activity-fault-tolerance.md).


 ## Supported scenarios
Copy Activity supports three scenarios for detecting, skipping, and logging incompatible data:

- **Incompatibility between the source data type and the sink native type**. <br/><br/> For example: Copy data from a CSV file in Blob storage to a SQL database with a schema definition that contains three INT type columns. The CSV file rows that contain numeric data, such as 123,456,789 are copied successfully to the sink store. However, the rows that contain non-numeric values, such as 123,456, abc are detected as incompatible and are skipped.
- **Mismatch in the number of columns between the source and the sink**. <br/><br/> For example: Copy data from a CSV file in Blob storage to a SQL database with a schema definition that contains six columns. The CSV file rows that contain six columns are copied successfully to the sink store. The CSV file rows that contain more or fewer than six columns are detected as incompatible and are skipped.
- **Primary key violation when writing to a relational database**.<br/><br/> For example: Copy data from a SQL server to a SQL database. A primary key is defined in the sink SQL database, but no such primary key is defined in the source SQL server. The duplicated rows that exist in the source cannot be copied to the sink. Copy Activity copies only the first row of the source data into the sink. The subsequent source rows that contain the duplicated primary key value are detected as incompatible and are skipped.

## Configuration
The following example provides a JSON definition to configure skipping the incompatible rows in Copy Activity:

```json
"typeProperties": {
    "source": {
        "type": "BlobSource"
    },
    "sink": {
        "type": "SqlSink",
    },         
    "enableSkipIncompatibleRow": true,           
    "redirectIncompatibleRowSettings": {
         "linkedServiceName": {
              "referenceName": "AzureBlobLinkedService",
              "type": "LinkedServiceReference"
            },
            "path": "redirectcontainer/erroroutput"
     }
}
```
Property | Description | Allowed values | Required
-------- | ----------- | -------------- | -------- 
enableSkipIncompatibleRow | Specifies whether to skip incompatible rows during copy or not. | True<br/>False (default) | No
redirectIncompatibleRowSettings | A group of properties that can be specified when you want to log the incompatible rows. | &nbsp; | No
linkedServiceName | The linked service of Azure Storage to store the log that contains the skipped rows. | The name of an AzureStorage or AzureStorageSas linked service, which refers to the storage instance that you want to use to store the log file. Azure Blob Storage and Azure Data Lake Storage are supported here. | No
path | The path of the log file that contains the skipped rows. | Specify the storage path that you want to use to log the incompatible data. If you do not provide a path, the service creates a container for you. | No

## Monitor skipped rows
After the copy activity run completes, you can see the number of skipped rows in the output of the copy activity:

```json
"output": {
            "dataRead": 95,
            "dataWritten": 186,
            "rowsCopied": 9,
            "rowsSkipped": 2,
            "copyDuration": 16,
            "throughput": 0.01,
            "redirectRowPath": "https://myblobstorage.blob.core.windows.net//myfolder/a84bf8d4-233f-4216-8cb5-45962831cd1b/",
            "errors": []
        },

```
If you configure to log the incompatible rows, you can find the log file at this path: `https://[your-blob-account].blob.core.windows.net/[path-if-configured]/[copy-activity-run-id]/[auto-generated-GUID].csv`. 

The log files can only be the csv files. The original data being skipped will be logged with comma as column delimiter if needed. We add two more columns "ErrorCode" and "ErrorMessage" in additional to the original source data in log file, where you can see the root cause of the incompatibility. The ErrorCode and ErrorMessage will be quoted by double quotes. 

An example of the log file content is as follows:

```
data1, data2, data3, "UserErrorInvalidDataValue", "Column 'Prop_2' contains an invalid value 'data3'. Cannot convert 'data3' to type 'DateTime'."
data4, data5, data6, "2627", "Violation of PRIMARY KEY constraint 'PK_tblintstrdatetimewithpk'. Cannot insert duplicate key in object 'dbo.tblintstrdatetimewithpk'. The duplicate key value is (data4)."
```

## Next steps
See the other Copy Activity articles:

- [Copy activity overview](copy-activity-overview.md)
- [Copy activity performance](copy-activity-performance.md)


