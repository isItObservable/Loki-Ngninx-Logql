# Is it Observable?
<p align="center"><img src="/image/logo.png" width="40%" alt="Prometheus Logo" /></p>

##  How to observe a Ngninx ingress controller?
### Part 2 using Loki and Logql
<p align="center"><img src="/image/loki_logo.png" width="40%" alt="Loki Logo" /></p>
Repository containing the files for the Episode 11 of Is it Observable : Observability of Nginx Controller using Loki


THis tutorial will reuse the steps describe in the Part 1 of our Tutorial.
Make sure to deploy :
- Nginx Ingress controller
- Prometheus Operator
- Configure the ingress controller for Grafana and the Google Hipster Shop
- Deploy the Google hipster Shop

### 1. Configure the ngninx controller
To generate logs containing extended information related to our ingress controller we need to update the configuration of ngninx to explose more logs.
let's modify the configmap to enhance the login format
```
kubectl edit cm nginx-config
```
add the following line:
```yaml
data:
  log_format: $remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent
    $request_time "upstreamAddress":"$upstream_addr", "upstreamResponseTime":"$upstream_response_time", "proxyHost":"$proxy_host", "upstreamStatus": "$upstream_status" "$resource_name" "$resource_type" "$resource_namespace" "$service"'
```

### 2. Install Loki
#### Install Loki with Promtail
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install loki grafana/loki-stack
```
#### Configure Grafana
In order to build a dashboard with data stored in Loki,we first need to add a new DataSource.
In grafana, goto Configuration/Add data source.
<p align="center"><img src="/image/addsource.PNG" width="60%" alt="grafana add datasource" /></p>
Select the source Loki , and configure the url to interact with it.

Remember Grafana is hosted in the same namesapce as Loki.
So you can simply refer the loki service :
<p align="center"><img src="/image/datasource.PNG" width="60%" alt="grafana add datasource" /></p>


### 3. Update the Grafana dashboard

#### explore the data provided by Loki in Grafana
In grafana select Explore on the main menu
Select the datasource Loki . IN the dropdow menu select the label produc -> hipster-shop
<p align="center"><img src="/image/explore.png" width="60%" alt="grafana explore" /></p>

#### Let's build a query
Loki has a specific query langage allow you to filter, transform the data and event plot a metric from your logs in a graph.
Similar to Prometheus you need to :
* filter using labels : {app="frontend",product="hipster-shop" ,stream="stdout"}
  we are here only looking at the logs from hipster-shop , app frontend and on the logs pushed in sdout.
* transform using |
  for example :
```
{namespace="hipster-shop",stream="stdout"} | json | http_resp_took_ms >10
```
the first ```|```  specify to Grafana to use the json parser that will extract all the json properties as labels.
the second ```|``` will filter the logs on the new labels created by the json parser.
In this example we want to only get the logs where the attribute http.resp.took.ms is above 10ms ( the json parser is replace . by _)

We can then extract on field to plot it using all the various [functions available in Grafana](https://grafana.com/docs/loki/latest/logql/)

if i want to plot the response time over time i could use the function :
```
rate({namespace="hipster-shop" } |="stdout" !="error" |= "debug" |="http.resp.took_ms" [30s])  ```