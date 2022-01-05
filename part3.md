# Is it Observable?
<p align="center"><img src="/image/logo.png" width="40%" alt="Prometheus Logo" /></p>

##  How to observe a Ngninx ingress controller?

### Part 3 using Log stream pipeline using Fluentd
<p align="center"><img src="image/Fluentd.png" width="40%" alt="Loki Logo" /></p>
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
  log_format: $remote_addr [$time_local] $request $status $body_bytes_sent $request_time $upstream_addr $upstream_response_time $proxy_host $upstream_status $resource_name $resource_type $resource_namespace $service
```
### 4. FluentD

#### 1. Generate the docker File

In order to deliver our tutorial we need a fluentd version having the following plugins preinstalled :
- the input plugin : forward ( to connect later on fluentbit with fluentd)
- the output plugin [dynatrace](https://github.com/dynatrace-oss/fluent-plugin-dynatrace)

To combine both plugins we are going to build the new image based from `fluentd-kubernetes-daemonset:v1.14.1-debian-forward-1.0`

To build the image you will need to require to install docker on your laptop : [docker desktop](https://www.docker.com/get-started)
```
cd /fluentd
docker build . -t fluentd_dynatrace_prometheus:0.1
```
The dockerfile only add the installation of the the library with this commad :
```
RUN gem install fluent-plugin-dynatrace -v 0.1.5
RUN gem install fluent-plugin-kubernetes_metadata_filter -v 2.9.1
RUN gem install fluent-plugin-multi-format-parser
RUN gem install fluent-plugin-concat
RUN gem install fluent-plugin-prometheus
```
In our tutorial i already have build the docker image and pushed it on docker hub.
We will use the following image : ```hrexed/fluentd_dynatrace_prometheus:0.1```

#### 2. Generate a Platfrom as a Service Token in Dynatrace
THe log ingest api of dynatrace is reachable only from the Active Gate.
To deploy the active Gate, it would be required to generate a Paas Token:
In dynatrace click :
* Settings
* Integration
* click on the button Generate
* Give a name and copy the value of the Paas Token
<p align="center"><img src="/image/paas.png" width="60%" alt="dt api scope" /></p>

#### 3. Generate API Token in Dynatrace
Follow the instruction described in [dynatrace's documentation](https://www.dynatrace.com/support/help/shortlink/api-authentication#generate-a-token)
Make sure that the scope log ingest is enabled.
<p align="center"><img src="/image/api_rigth.png" width="60%" alt="dt api scope" /></p>

#### 4. Get the cluster id of your K8s cluster
```
kubectl get namespace kube-system -o jsonpath='{.metadata.uid}'
```
#### 5. update the deployment of fluend and of the active gate

* Create a service account and cluster role for accessing the Kubernetes API.
```
kubectl apply -f fluentd/service_account.yaml
```
Create a secret holding the environment URL and login credentials for this registry, making sure to replace.
```
export ENVIRONMENT_URL=<with your environment URL (without 'http'). Example: environment.live.dynatrace.com>
export CLUSTERID=<YOUR CLUSTER ID>
export PAAS_TOKEN=<YOUR PAAS TOKEN>
export API_TOKEN=<YOUR API TOKEN>
export ENVIRONMENT_ID=<YOUR environementid in your environment url>
kubectl create secret docker-registry tenant-docker-registry --docker-server=${ENVIRONMENT_URL} --docker-username=${ENVIRONMENT_ID} --docker-password=${PAAS_TOKEN} -n dynatrace
kubectl create secret generic tokens --from-literal="log-ingest=${API_TOKEN}" -n dynatrace
 ```

Update the file named fluentd/fluentd-manifest.yaml and activegate.yaml, by running the following command :
 ```
sed -i "s,ENVIRONMENT_ID_TO_REPLACE,$ENVIRONMENT_ID," fluentd/fluentd-manifest.yaml
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," fluentd/fluentd-manifest.yaml
sed -i "s,ENVIRONMENT_URL_TO_REPLACE,$ENVIRONMENT_URL," fluentd/activegate.yaml
sed -i "s,ENVIRONMENT_ID_TO_REPLACE,$ENVIRONMENT_ID," fluentd/fluentd-configmap-dynatrace.yaml
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," fluentd/fluentd-configmap-dynatrace.yaml
 ```

#### 6. Deploy
```
kubectl apply -f fluentd/activegate.yaml
kubectl apply -f fluentd/fluentd-manifest.yaml
```

#### 7. Connect the active Gate to your dynatrace tenant
To get native Kubernetes metrics, you need to connect the Kubernetes API to Dynatrace.

Get the Kubernetes API URL.
```

kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```
Get the bearer token from the dynatrace-monitoring service account.
```
kubectl get secret $(kubectl get sa dynatrace-monitoring -o jsonpath='{.secrets[0].name}' -n dynatrace) -o jsonpath='{.data.token}' -n dynatrace | base64 --decode
```
In the Dynatrace menu, go to Settings > Cloud and virtualization > Kubernetes, and select Connect new cluster.
Provide a Name, Kubernetes API URL, and the Bearer token for the Kubernetes cluster. Note: For Rancher distributions, you need the bearer token that was created in Rancher web UI, as described in Special instructions for Rancher distributions above. Once you connect your Kubernetes clusters to Dynatrace, you can get native Kubernetes metrics, like request limits, and differences in pods requested vs. running pods.


#### 8. Let's have a look at the fluentd pipeline producing prometheus metrics
The first important part is to collect the logs and extract all the relevant medata data.
This would be done using the tail plugin :
```
 <source>
      @type tail
      path /var/log/containers/*nginx*.log
      pos_file /var/log/fluentd.pos
      time_format %Y-%m-%dT%H:%M:%S.%NZ
      tag nginx
      <parse>
        @type nginx
        key_name log
        reserve_data yes
        expression  /^(?<logtime>\S+)\s+(?<logtype>\S+)\s+(?<type>\w+)\s+(?<ip>\S+)\s+\[(?<time_local>[^\]]*)\]\s+(?<method>\S+)\s+(?<request>\S+)\s+(?<httpversion>\S*)\s+(?<status>\S*)\s+(?<bytes_sent>\S*)\s+(?<responsetime>\S*)\s+(?<proxy>\S*)\s+(?<upstream_responsetime>\S*)\s+(?<ressourcename>\S*)\s+(?<upstream_status>\S*)\s+(?<ingress_name>\S*)\s+(?<ressource_type>\S*)\s+(?<ressource_namesapce>\S*)\s+(?<service>\w*)/
        types ip:string,time_local:string,method:string,request:string,httpversion:string,status:integer,bytes_sent:integer,responsetime:float,request_time:float,proxy:string,upstream_responsetime:float,ressourcename:string,ressource_type:string,ressource_namesapce:string,service:string
        time_format %d/%b/%Y:%H:%M:%S %z
      </parse>
      read_from_head true
      keep_time_key true
    </source>
```

the operator `expression` and `types` will extact the various information of our nginx log

then we need to initialize the prometheus exporter: 
```
<source>
       @type prometheus
       bind 0.0.0.0
       port 9914
       metrics_path /metrics
</source>
```

and create the various metrics that we would like to expose :
```
<filter  nginx>
      @type prometheus
      <labels>
        method ${method}
        request ${request}
        status ${status}
        ingressnamespace ${ressource_namesapce}
        targetservice ${service}
        ressourcename ${ressourcename}
      </labels>
      <metric>
        name ingress_response_time
        type gauge
        desc responset time
        key responsetime
      </metric>
      <metric>
        name ingress_byte_sent
        type gauge
        desc byte sent
        key bytes_sent
      </metric>
      <metric>
        name ingress_requests
        type counter
        desc The total number of request
      </metric>
      <metric>
        name ingress_status
        type counter
        desc status code
        key status
      </metric>
    </filter>
```

To debug the extacted data, i recommend to use the `stdout` output plugin to validate your extractions.

#### 9. Expose the new metrics to Prometheus
To let the prometheus operator scrape new exporter, we need to deploy a SerivceMonitor.
The ServiceMonitor will refer to a service. So let's first create a service pointing our Fluentd pods 
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: fluentd-exporter
  name: fluentd-prom-metrics
  annotations:
    prometheus.io/scrape: 'true'
    prometheus.io/scheme: 'http'
    prometheus.io/port: '9914'
    prometheus.io/path: '/metrics'
    metrics.dynatrace.com/port: '9914'
    metrics.dynatrace.com/scrape: 'true'
    metrics.dynatrace.com/path: '/metrics'
spec:
  ports:
    - port: 9914
      name: fluentdprom
      targetPort: fluentdprom
      protocol: TCP
  selector:
    app: fluentd-pipeline
```
Then we can create the ServiceMonitor:
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: fluentd-servicemon
  labels:
    app: fluentd-pipeline
    release: prometheus
spec:
  endpoints:
    - interval: 5s
      port: fluentdprom
  selector:
    matchLabels:
      app: fluentd-exporter
```

#### 9. Let's build a dashboard
By adding the Prometheus and Dynatrace annotation on our service, the generated metrics are ingested by Prometheus and dynatrace.
Let's first enhace our grafana dashboard

#### Request/s
Each line of generated by nginx corresponds to a http request.
There fore to graph the number of request /s we can build the following promql
```
sum by (targetservice) (rate(ingress_requests{targetservice!=""}[30s]))
```
#### bytes sent
Becase we are able to generate a Gauge metric with the byte sent we can use the following promql to graph the byte_sent per services
```
avg by (targetservice) (ingress_byte_sent{targetservice!=""})
```
#### Response time
```
avg by(targetservice) (ingress_response_time{targetservice!=""})
```
#### 10. update the fluentd pipeline to push metrics and logs to Dynatrace

In order to ingest the logs in dynatrace , we would need to use modify our generated log stream
using `record_transformer`

We need to have a content field with some data and add kubernetes metadata : 
```
 <filter nginx>
      @type record_transformer
      enable_ruby true
        <record>
          status ${ record.dig(:log, :severity) || record.dig(:log, :level) || (record["log"] =~ /\W?\berror\b\W?/i ? "ERROR" : (record["log"] =~ /\W?\bwarn\b\W?/i ? "WARN" : (record["log"] =~ /\W?\bdebug\b\W?/i ? "DEBUG" : (record["log"] =~ /\W?\binfo\b\W?/i ? "INFO" : "NONE")))) }
          content ${record["method"]} ${record["request"]} ${record["status"]} ${record["service"]} ${record["bytes_sent"]} ${record["responsetime"]} ${record["service"]}
          dt.kubernetes.node.system_uuid ${File.read("/sys/devices/virtual/dmi/id/product_uuid").strip}
          dt.kubernetes.cluster.id "#{ENV['CLUSTER_ID']}"
          k8s.namespace.name ${record["ressource_namesapce"]}
          k8s.service.name ${record["service"]}
        </record>
        remove_keys  nginx
    </filter>
```
Then we simply need to use the dynatrace plugin:*
```
<match nginx>
@type              dynatrace
active_gate_url "#{ENV['AG_INGEST_URL']}"
api_token "#{ENV['LOG_INGEST_TOKEN']}"
ssl_verify_none    true
</match>
```

let's apply our updated version of the fluentd configmap 
```
kubectl apply -f fluentd/fluentd-configmap-dynatrace.yaml -n dynatrace
```
To force fluentd to take the new configuration , we need to delete the existing pods :
```
kubectl delete pods -n dynatrace -l app=fluentd-pipeline
```

#### 11. Let's visualize the data in Dynatrace

##### Prometheus metrics


##### Create a metric from the generated logs





