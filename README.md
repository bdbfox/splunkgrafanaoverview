<!-- $theme: default -->

<!-- $size: 16:9 -->
# Splunk and Grafana Overview

Follow along at:

https://github.com/bdbfox/splunkgrafanaoverview

---

# Splunk Basics

splunk.admin.foxdcg.com (http://splunk.admin.foxdcg.com/)

Start with the “Search and Reporting” app. Unless you are an admin, most everything you need to do will be in there.

If you don't have access contact, contact Brian DeBoer, Fabian Utomi or Shakeel Sorathia

---

### Select the index to search against

* API service log data is in dcg_prod, dcg_stage, etc...
  `index="dcg_prod"`
* If you can't remember what indices there are:
```
| eventcount summarize=false index=* 
  | dedup index | fields index
```

---

### Basic search guidelines

* Generally, you need quotes around phrases and field values that include white spaces, commas, pipes, quotes, or brackets.
* You can do wildcard with an asterisk, this can be before or after, ex. something* vs *something.
* If you need to you can backslash ( \\ ) asterisks or quotes to search for those characters.
* You can use AND, NOT and OR, and group queries with (), AND is implied.
* For numeric values you can use >, <, >=, <= etc...
* != is not the same as NOT. != will hide results that are missing the field name.
* Keep the time range at a reasonable limit to optimize your queries.
* Use realtime search to look for data as it comes in.

---

### Log Format

* Logs are formatted as space separated key value pairs. By doing it in that format, you get automatic field separation when you format your logs in that way.
* Data in logs that includes spaces or special characters should be wrapped in quotes, ie. request paths.
* Data that you'd like to be treated as numeric (to be able to do ranges queries for) should not be wrapped in a quote.

---

### Important (Automatic) fields

* `attrs.SERVICE_NAME` - the name of the service
* `attrs.SERVICE_VERSION` - the version of the service, ie. v1, v2, etc...
* `rt` - Response Time in ms
* `os` - Offset from request start in ms
* `method` - The request method, ie. GET, POST, PUT
* `dma` - The dma that the request came through
* `handler` - The “controller” or action that processed the request
* `rid` - request id - A unique identifier for every incoming request
* `level` - The log level, ie. info, warn, error
* `path` - The request path
* `sc` - Status code. ie 200, 404, 500
* `t` - The ISO8601 timestamp of the log statement

---

### Useful queries

* Search for all logs from the screens service
  `index=dcg_prod attrs.SERVICE_NAME=screens`
* Look up all services
  `index="dcg_prod" | top 0 attrs.SERVICE_NAME`
* Find all the events from a particular request id
* Limit logs from the green deployment
* Search for all errors from a particular service
* Group statistics with the “top” command

---

## Grafana Basics

grafana.admin.foxdcg.com (http://grafana.admin.foxdcg.com/)

If you don't have access contact, contact Brian DeBoer, Fabian Utomi or Shakeel Sorathia

---

### Useful Dashboards

* Client Dashboard
* Cloudwatch
* Generic {LanguageName} Dashboards
* Alerts
* Linkerd, consul, ECS, etc... dashboards

---

### Client Dashboard
Provides an overview of requests by apikey, broken down by service.

* Total requests by apikey
* Total requests by service
* Requests by service / by apikey

---

### Cloudwatch
Includes data directly from AWS Cloudwatch for things like API Gateway and FOX Profile. Useful for checking the toplevel health of the APIs.

* API Gateway Requests
* 5xx error count totals and by endpoint
* API Gateway Latency
* API Gateway Requests by endpoint
* Redis health and stats
* FOX Profile latency, requests and health

---

### Generic {LanguageName} Dashboards
The overview of services built in that specific language. From the perspective of the service itself.

* CPU usage by service
* Memory usage by service
* Response time by service
* Error count by service
* Total requests by service and requests by apikey
* Loop delay (in node)
* Geo lookup and other method timings
 
---

### Alerts
Grouped together alerts. Because alerts don't support templating some alerts must have their own graphs.

---

### Linkerd, Consul, ECS, etc... dashboards
More detailed information on the infrustructure

* Linkerd will show a good overview on throughput of the services and overall success/failure rate by service.
* Consul dashboard will provide you insight into the overall health of consul.
* ECS System Usage will provide insight into the number of hosts and utilization of the container instances.

---

# Splunk Advanced Topics

---

## Creating a dashboard

1. Click on "Dashboards" in top bar.
2. Click on "Create new Dashboard".
3. Name your dashboard and set the permissions settings accordingly.

---


### Add your first panel

#### Add a table of requests by status code and method

1. Click "Add Panel".
2. Choose New -> Statistics Table.
3. Choose a reasonable time range.
4. Write your query. Try and limit your data if you can.
```
index="dcg_prod" path=* | stats count by sc, method | sort count desc
```

---

### Convert your panel to use inputs

#### Add a index filter

1. Click "Add Input".
2. Choose Dropdown.
3. Give it a label (optional).
4. Click "Search on Change".
5. Give it a memorable token name. You will reference this as `${tokenName}$` in your queries.
6. Make it dynamic or static. If you choose to use a dynamic filter put in the search string and choose fields for label and value. A good dynamic list for indices query is:
  ```
  | eventcount summarize=false index=* | dedup index | fields index
  ```
7. Choose defaults.

---

#### Add a time range filter

1. Click "Add Input".
2. Choose Time.
3. Give it a label (optional).
4. Click "Search on Change".
5. Give it a memorable token name. You will reference this as `${tokenName}$` in your queries.
6. Give it a reasonable default.

---

#### Change your panel to use these

1. Adjust the query to change the index to use the token.
`index=$index_token$`
2. Adjust the time range to use the "Shared Time Picker".

##### Note: Sometimes splunk gets cranky. Try clicking on "Source" and then back to "UI" to resolve.
--- 

### Create a response time pie chart

1. Click on "Add Panel"
2. Choose New -> Pie Chart
3. Select Shared Time Picker
4. Enter your query and setup the ranges
```
index=$index_token$ path=*
  | rangemap field=rt "<0.5"=0-50 "<1"=0-100 "<1.5"=0-150 "<2.0"=0-200 default=">2.0"
  | stats count as "Number of Transactions" by range
```

---

### Create a list of top apikeys

1. Click on "Add Panel"
2. Choose New -> Statistics Table
3. Select Shared Time Picker
4. Enter your query
```
index=$index_token$ path!=* | top apikey
```

###### Adjust your apikey list to be by service

```
index=$index_token$ path!="/service/health*" | stats count by "attrs.SERVICE_NAME", apikey | sort count desc
```

---

### Create a geo location cluster chart

1. Click on "Add Panel"
2. Choose New -> Cluster Map
3. Select Shared Time Picker
4. Create your query
```
index=$index_token$ "attrs.SERVICE_NAME"=screens method=GET geoLongitude
  | rex "geoLongitude=\+?(?<longitude>[0-9.-]*)"
  | rex "geoLatitude=\+?(?<latitude>[0-9.-]*)"
  | geostats latfield=latitude longfield=longitude count
```

---

### Lookup Tables

TBD

---

# Grafana Advanced Topics

---

## Creating a dashboard in Grafana

1. Click on the menu -> Dashboards and select "New"
2. Choose a panel type, Graph
3. Click on the title of the panel to get to the Edit screen
4. Choose your data source and write your query

More details to come



