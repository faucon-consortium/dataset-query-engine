## DATASET QUERY ENGINE

The Query Engine is a java module allowing to perform dynamic (search) queries on the FAUCON Use Cases Datasets.

Those various datasets are stored (indexed) in ElastiSearch. By this way, we can perform many and various data manipulating and analysis operations.

The Datasets are *(many more to come !)*:

- [Sensulogs](https://github.com/faucon-consortium/sensu-log-model)
  SMILE Cloud Computing Integration platform Sensu supervision log files
- Social Twitter tweets data    
-----------------------------

The query request is formalized through a JSON message and look like the SQL syntax  
Here is the JSON Schema:

```
 root
 |-- select: array (nullable = false)
 |    |-- element: string (containsNull = false)
 |-- from: string (nullable = false)
 |-- where: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- clause: struct (nullable = true)
 |    |    |    |-- parenthesis: string (nullable = true)
 |    |    |    |-- condition: struct (nullable = true)
 |    |    |    |    |-- index: string (nullable = true)
 |    |    |    |    |-- field: string (nullable = true)
 |    |    |    |    |-- operator: string (nullable = true)
 |    |    |    |    |-- value: string (nullable = true)
 |    |    |    |-- logicalOp: string (nullable = true)
 |-- groupBy: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- missing: boolean (nullable = true)
 |    |    |-- name: string (nullable = true)
 |    |    |-- order: string (nullable = true)
 |-- orderBy: array (nullable = true)
 |    |-- element: string (containsNull = true)
 |-- prettyPrint: boolean (nullable = true)
 |-- scrollID: string (nullable = true)
 |-- searchParams: struct (nullable = false)
 |    |-- nbAggregationsToRetrieve: long (nullable = true)
 |    |-- nbDocumentsToRetrieve: long (nullable = true)
 |    |-- nbResultsToRetrieve: long (nullable = false)
 |    |-- scrollSearchTimeout: long (nullable = true)

```
-----------------------------
### Fields Description:
#### -- select: array of String
This field, list the data elements to retrieve from a dataset collection. The item name data is expressed by its **full JSON name**. This means that the name of the field corresponds to the **full path access** to the field with *each tree level* separated by a point.  

##### Example: #####
```
"select": ["id", "server.name"]
```
Wildcards can be used to build the field name:
```
"select": ["server.*"]
or
"select": ["*"]
```
The fisrt example aims to retrieve **all recursive childs elements** from the *root element*: ***server*** and the last one means retrieving all data from the JSON root.  


There is a special syntax that is used to express data aggregations. The supported aggregations are: COUNT, SUM, AVG, MAX and MIN

Here are some examples:

- **```"select" : ["MAX(customer.age) AS Age"]```**
   Get the maximum customer age value. The "AS" clause (case sensitive) allow to define the name of the field that will be reported for this aggregation in the result JSON document.
- **```"select" : ["MIN(customer.age)"]```**
   Get the minimum customer age value. As there is no "AS" clause, the name of the field returned will be ```"MIN_"<full field name>```.  
   In the example above, the name of the field returned will be: ```"MIN_customer.age"```
- **```"select" : ["SUM(customer.age) AS TotalAge"]```**
   Get the summation of all customers ages  
   
- **```"select" : ["COUNT(customer.age)"]```**
   Get the number of customer ages
    
- **```"select" : ["AVG(customer.age)"]```**
   Get the average customer age   


#### -- from: String
This field define the index name (look like a Table in SQL) on wich the requests are performed  


#### -- where: List<Clause>
This field define a list of filters, applied to the data retreival. 
##### Parameters definition: #####
  + <ins>parenthesis</ins>: STRING
  This field take two values, **"START"** to open a parenthesis "(" and **"END"** to close the parenthesis ")"
  + <ins>condition</ins>
  This field define the filter conditions applied to the data fields
    + <ins>index</ins>: STRING
    The name of the index on wich perform the filter
    + <ins>field</ins>: STRING
    The full name of the field on wich the condition applied
    + <ins>operator</ins>: STRING
    Define the operator condition. The following operators are supported:
      + "="
      + "!="
      + "<"
      + "<="
      + ">"
      + ">="
      + "[]" (range of data with bounds included)
      + "{}" (range of data with bounds excluded)
      + <ins>value</ins>
    Define the field value of the condition. There is a special value: "NULL" to express the nullity of a field.
    + <ins>logicalOp</ins>
  Define the logical operator used to *link* two conditions. Two values are supported: "AND" and "OR" logical operator.  
  
  
  

For example, the below sample can be expressed in *SQL Syntax* as : ```"where customer.dob IS NOT NULL AND (customer.civility="Mme" OR customer.civility="M")```
```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "customers",
        "field" : "customer.dob",
        "operator" : "!=",
        "value" : "NULL"
      },
      "logicalOp" : "AND"
    }
  }, {
    "clause" : {
      "parenthesis" : "START",
      "condition" : {
        "index" : "customers",
        "field" : "customer.civility",
        "operator" : "=",
        "value" : "Mme"
      },
      "logicalOp" : "OR"
    }
  }, {
    "clause" : {
      "parenthesis" : "END",
      "condition" : {
        "index" : "customers",
        "field" : "customer.civility",
        "operator" : "=",
        "value" : "M"
      }
    }
  } ]
```
##### Dates condition: #####
All the dates are stored in format **"yyyy-MM-dd'T'HH:mm:ss.SSSSSS"** with:
+ ```yyyy```: Year in four digits (ex: 2020)
+ ```MM```: The month number from the year, in two digits (ex: 09, ex:11)
+ ```dd```: The day number from the month, in two digits (ex: 02, ex:31)
+ ```HH```: Hour in 24h format (ex:09, ex:17)
+ ```mm```: Minutes (ex:23, ex:47)
+ ```ss```: Seconds (ex:05, ex:33)
+ ```SSSSSS```: Milli seconds in 6 digits (ex:213070)

Example:
2020-02-10T06:26:10.213070  
  
You can query the date fields in different format:
+ By Year (```yyyy```): Get all documents from the same year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020}"
      }
    }
  } ]
  ```
+ By Month (```yyyy-MM```): Get all documents from the same month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02"
      }
    }
  } ]
  ```
+ By Day (```yyyy-MM-dd```): Get all documents from the same day, month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29"
      }
    }
  } ]
  ```
+ By Hour (```yyyy-MM-ddTHH```): Get all documents from the same hour, day, month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29T06"
      }
    }
  } ]
  ```
+ By Minutes (```yyyy-MM-ddTHH:mm```): Get all documents from the same minute, hour, day, month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29T06:25"
      }
    }
  } ]
  ```

+ By Seconds (```yyyy-MM-ddTHH:mm:ss```): Get all documents from the same second, minute, hour, day, month and year
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29T06:25:17"
      }
    }
  } ]
  ```
+ By Milliseconds (```yyyy-MM-ddTHH:mm:ss.SSSSSS```): Get all documents from the same DateTime
  ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "2020-02-29T06:25:17.213070"
      }
    }
  } ]
  ```

+ By Range:
  You can specify a date range like that:
   ```
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "timestamp",
        "operator" : "=",
        "value" : "[2020-02-29T06,2020-02-29T08}"
      }
    }
  } ]
  ```
The **[** and **]** characters defines **inclusive bounds**. On the other side, the **{** and **}** characters defines **exclusive bounds**.


#### -- groupBy: List<GroupBy>
Defines the fields to be grouped for the calculation of aggregations.
##### Paremeters Definition: #####

  + <ins>name</ins>: String
  The name of the field to be grouped
  + <ins>missing</ins>: Boolean
  If true an explicit null bucket will represent documents with missing values.
  + <ins>order</ins>: String
  Define the *groupBy* sort order: "ASC" for ascending sort and "DESC" for descending sort

##### Example: #####
 Group by customer civility in ascending order (first:"M" and second:"Mme") and for each civility, group by descending customer age
 ```
 "groupBy" : [ {
    "name" : "customer.civility",
    "order" : "ASC",
    "missing" : true
  }, {
    "name" : "cutomer.age",
    "order" : "DESC",
    "missing" : true
  } ],
```

Can be expressed in *SQL Syntax*  as : ```"GROUP BY customer.civility,customer.age```


#### -- orderBy: List<String>
Define the *general (or extended)* sort order.
The syntax is: "<*Sort Order*> (*data field fullname*)". "Sort Order" can take two values: "ASC" for ascending sort and "DESC" for descending sort.

##### Example: #####
```
"orderBy" : [ "ASC (customer.nbVisit)" ]
```

#### -- prettyPrint: Boolean
If set to **true**, the JSON results message is retuned as a formated string whith indent and line break. If set to **false**, the JSON message is compacted in one line.


#### -- scrollID: String
When the amount of data retreival is very large, you can enable the *pagination mode*. In this mode, the results are split by *blocks*, each block with a defined number of recordset results.
The **scrollID** represent the *current* cursor position in the dataset.
This field is automaticly generated by the API. You have just to return it to retrieve the following block results.

#### -- searchParams: Struct
Define how and many results would be retreive.
##### Data Defintion: #####
  + <ins>nbDocumentsToRetrieve</ins>: Long
  The number of documents to retrieve. If set to **-1**, all the filtered documents are retrieve
  + <ins>nbAggregationsToRetrieve</ins>: Long
  The number of aggregations to retrieve. If set to **-1**, all the defined aggregations are retrieve
  + <ins>nbResultsToRetrieve</ins>: Long
  Define the number of recordset retrieve by block. It is recommended to activate the pagination mode. 
  + <ins>scrollSearchTimeout</ins>: Long
  Define, in time second, the scroll living timeout. When it is reached, the cursor is destroyed.

##### Example: #####
```
"searchParams" : {
    "nbResultsToRetrieve" : 2,
    "nbDocumentsToRetrieve" : -1,
    "nbAggregationsToRetrieve" : 5,
    "scrollSearchTimeout" : 600
  }
```
-----------------------------

### Results:
Two format of results are returned:

+ document
The returned result is a list of JSON messages compound and filtred by the query definition.
That's mean, that the **return JSON message varies** because it is composed of only the fields defined by the "select" expression, their values filtered by the "where" expression.

Here is for example, the JSON schema for a full JSON Sensu Log result:

```
root
 |-- documents: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- event: struct (nullable = true)
 |    |    |    |-- action: string (nullable = true)
 |    |    |    |-- check: struct (nullable = true)
 |    |    |    |    |-- executed: string (nullable = true)
 |    |    |    |    |-- history: array (nullable = true)
 |    |    |    |    |    |-- element: string (containsNull = true)
 |    |    |    |    |-- issued: string (nullable = true)
 |    |    |    |    |-- name: string (nullable = true)
 |    |    |    |    |-- output: string (nullable = true)
 |    |    |    |    |-- status: long (nullable = true)
 |    |    |    |    |-- thresholds: struct (nullable = true)
 |    |    |    |    |    |-- critical: long (nullable = true)
 |    |    |    |    |    |-- warning: long (nullable = true)
 |    |    |    |    |-- total_state_change: long (nullable = true)
 |    |    |    |    |-- type: string (nullable = true)
 |    |    |    |-- client: struct (nullable = true)
 |    |    |    |    |-- address: string (nullable = true)
 |    |    |    |    |-- name: string (nullable = true)
 |    |    |    |    |-- subscriptions: array (nullable = true)
 |    |    |    |    |    |-- element: string (containsNull = true)
 |    |    |    |    |-- timestamp: string (nullable = true)
 |    |    |    |    |-- version: string (nullable = true)
 |    |    |    |-- id: string (nullable = true)
 |    |    |    |-- last_ok: string (nullable = true)
 |    |    |    |-- last_state_change: string (nullable = true)
 |    |    |    |-- occurrences: long (nullable = true)
 |    |    |    |-- occurrences_watermark: long (nullable = true)
 |    |    |    |-- silenced: boolean (nullable = true)
 |    |    |    |-- silenced_by: array (nullable = true)
 |    |    |    |    |-- element: string (containsNull = true)
 |    |    |    |-- timestamp: string (nullable = true)
 |    |    |-- handler_name: string (nullable = true)
 |    |    |-- id: string (nullable = true)
 |    |    |-- level: string (nullable = true)
 |    |    |-- message: string (nullable = true)
 |    |    |-- payload: struct (nullable = true)
 |    |    |    |-- command: string (nullable = true)
 |    |    |    |-- handlers: array (nullable = true)
 |    |    |    |    |-- element: string (containsNull = true)
 |    |    |    |-- issued: string (nullable = true)
 |    |    |    |-- name: string (nullable = true)
 |    |    |    |-- source: string (nullable = true)
 |    |    |-- subscribers: array (nullable = true)
 |    |    |    |-- element: string (containsNull = true)
 |    |    |-- timestamp: string (nullable = true)
 |-- searchQueryParams: struct (nullable = true)
 |    |-- index: string (nullable = true)
 |    |-- nbResultsToRetrieve: long (nullable = true)
 |    |-- totalResults: long (nullable = true)
```
+ aggregation

The returned result is a list of JSON Aggregation messages compound by the "select <aggregation>", group by the "groupBy" expression and filtred by the query "where" definition.


Here is the JSON schema for an aggregation result. It's alawys the same schema:
```
root
 |-- aggregations: struct (nullable = true)
 |    |-- Global: array (nullable = true)
 |    |    |-- element: struct (containsNull = true)
 |    |    |    |-- name: string (nullable = true)
 |    |    |    |-- value: string (nullable = true)
 |    |-- GroupBy: array (nullable = true)
 |    |    |-- element: struct (containsNull = true)
 |    |    |    |-- agg: struct (nullable = true)
 |    |    |    |    |-- docCount: long (nullable = true)
 |    |    |    |    |-- keys: array (nullable = true)
 |    |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |    |-- name: string (nullable = true)
 |    |    |    |    |    |    |-- value: string (nullable = true)
 |    |    |    |    |-- aggregations: array (nullable = true)
 |    |    |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |    |    |-- name: string (nullable = true)
 |    |    |    |    |    |    |-- value: string (nullable = true)
 ```
##### Data Defintion: #####
  + <ins>Global</ins>: Struct
  The global value for all aggrated data
    + name: String
    The name of the aggregation. It's correpond to the "select aggregation" definition (name build with "AS or generated with not")
    + value:
    The global aggregation value computed on all selected documents
  
  ###### Example: ######
  ```
     "Global" : [ {
      "name" : "NbMessageByName",
      "value" : "767804.0"
    } ]
  ```
  
  + <ins>GroupBy</ins>: List<Agg>
  The list of all aggregations computed
    + docCount: Long
    The total number of all selected documents compute from the "select"  and "groupBy" expressions.
    + keys:
    The list of all terms name and values build from the "groupBy" expression
      + name: String
      The name of the "groupBy" fields
      + value: String
      The value associated to a "groupBy" field
      
 ###### Example: ######
  ```
     "agg" : {
        "docCount" : 20,
        "keys" : [ {
          "name" : "message",
          "value" : "completing work in progress"
        } ],
        "aggregations" : [ {
          "name" : "NbMessageByName",
          "value" : "20.0"
        } ]
      }
    },...
 ```
 -----------------------------
 ### Queries Examples:

+ <ins>Document Query</ins>:
   Query SQL look like:
   ```select * from sensulogs where id="2020-02-20T06:25:45.468613" OR id="2020-02-20T06:27:14.298428"```
  + Query:
  ```
  {
  "select" : [ "*" ],
  "from" : "sensulogs",
  "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "id",
        "operator" : "=",
        "value" : "2020-02-20T06:25:45.468613"
      },
      "logicalOp" : "OR"
    }
  }, {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "id",
        "operator" : "=",
        "value" : "2020-02-20T06:27:14.298428"
      }
    }
  } ],
  "orderBy" : [ "ASC (id)" ],
  "prettyPrint" : true,
  "searchParams" : {
    "nbResultsToRetrieve" : 2,
    "nbDocumentsToRetrieve" : 10,
    "nbAggregationsToRetrieve" : 0,
    "scrollSearchTimeout" : 600
   }
  }
  ```
  
  + Results:
  ```
  {
  "searchQueryParams" : {
    "index" : "sensulogs",
    "nbResultsToRetrieve" : -7,
    "totalResults" : 3
  },
  "documents" : [ {
    "id" : "2020-02-20T06:25:45.468613",
    "event" : {
      "action" : "create",
      "check" : {
        "executed" : "2020-02-20 06:25:45",
        "history" : [ "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2", "2" ],
        "issued" : "2020-02-20 06:25:45",
        "name" : "keepalive",
        "output" : "No keepalive sent from client for 1727039 seconds (>=180)",
        "status" : 2,
        "thresholds" : {
          "critical" : 180,
          "warning" : 120
        },
        "total_state_change" : 0,
        "type" : "standard"
      },
      "client" : {
        "address" : "10.10.10.242",
        "name" : "seyia.vitry.intranet",
        "subscriptions" : [ "openvz", "client:seyia.vitry.intranet" ],
        "timestamp" : "2020-01-31 06:41:46",
        "version" : "1.6.2"
      },
      "id" : "8bcfb6ed-011a-409f-b399-d70d1f673bfa",
      "last_ok" : "2020-01-31 06:42:21",
      "last_state_change" : "2020-01-31 07:22:18",
      "occurrences" : 56105,
      "occurrences_watermark" : 56105,
      "silenced" : false,
      "silenced_by" : [ ],
      "timestamp" : "2020-02-20 06:25:45"
    },
    "level" : "info",
    "message" : "processing event",
    "timestamp" : "2020-02-20T06:25:45"
  }, {
    "id" : "2020-02-20T06:25:45.468729",
    "handler_name" : "default",
    "level" : "error",
    "message" : "unknown handler",
    "timestamp" : "2020-02-20T06:25:45"
  }, {
    "id" : "2020-02-20T06:27:14.298428",
    "level" : "info",
    "message" : "publishing check request",
    "payload" : {
      "command" : "check-http.rb -t 30 -u https://hubgr.groupe-rocher.com/fr/login  -w",
      "handlers" : [ "mattermost" ],
      "issued" : "2020-02-20 06:27:14",
      "name" : "rocher_web_site",
      "source" : "rocher"
    },
    "subscribers" : [ "rocher" ],
    "timestamp" : "2020-02-20T06:27:14"
   } ]
  }
  ```
 
+ <ins>Aggregations Query</ins>:
  Query SQL look like:
   ```select count(message) from sensulogs order by message asc```
  + Query:
  ```
  {
   "select" : [ "COUNT(message) AS NbMessageByName" ],
   "from" : "sensulogs",
   "where" : [ {
    "clause" : {
      "condition" : {
        "index" : "sensulogs",
        "field" : "message",
        "operator" : "!=",
        "value" : "NULL"
      }
    }
   } ],
   "groupBy" : [ {
    "name" : "message",
    "order" : "ASC",
    "missing" : true
   } ],
   "prettyPrint" : true,
   "searchParams" : {
    "nbResultsToRetrieve" : 2,
    "nbDocumentsToRetrieve" : 0,
    "nbAggregationsToRetrieve" : -1,
    "scrollSearchTimeout" : 600
   }
  }
  ```
 
 
 + Result (limited for clearity):
 ```
 {
  "aggregations" : {
    "Global" : [ {
      "name" : "NbMessageByName",
      "value" : "767804.0"
    } ],
    "GroupBy" : [ {
      "agg" : {
        "docCount" : 20,
        "keys" : [ {
          "name" : "message",
          "value" : "completing work in progress"
        } ],
        "aggregations" : [ {
          "name" : "NbMessageByName",
          "value" : "20.0"
        } ]
      }
    }, {
      "agg" : {
        "docCount" : 279,
        "keys" : [ {
          "name" : "message",
          "value" : "config file applied changes"
        } ],
        "aggregations" : [ {
          "name" : "NbMessageByName",
          "value" : "279.0"
        } ]
      }
    },{
      "agg" : {
        "docCount" : 187066,
        "keys" : [ {
          "name" : "message",
          "value" : "processing event"
        } ],
        "aggregations" : [ {
          "name" : "NbMessageByName",
          "value" : "187066.0"
        } ]
      }
     }
    ]
  }
}
 ```

 -----------------------------
 ### Build:
```
mvn clean install
```

