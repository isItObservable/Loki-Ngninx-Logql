# Is it Observable?
<p align="center"><img src="/image/logo.png" width="40%" alt="Prometheus Logo" /></p>

##  How to observe a Ngninx ingress controller using Loki and Logql
<p align="center"><img src="/image/loki_logo.png" width="40%" alt="Loki Logo" /></p>
Repository containing the files for the Episode 11 of Is it Observable : Observability of Nginx Controller using Loki


This repository showcase the usage of the Loki  by using GKE with :
- the HipsterShop


## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud containr clusters create isitobservable \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone Github repo
```
git clone https://github.com/isItObservable/Episode2--Kubernetes-Loki

```
### 4. Install the ngninx controller
#### 1. Deploy
```
helm repo add nginx-stable https://helm.nginx.com/stable
helm install ngninx nginx-stable/nginx-ingress --set controller.enableLatencyMetrics=true, prometheus.create=true,controller.config.name=nginx-config
```
this command will install the nginx controller with configmap named nginx-config




#### 2. configure
To generate logs containing extended information related to our ingress controller we need to update the configuration of ngninx to explose more logs.
let's modify the configmap to enhance the login format
```
kubectl edit cm nginx-config
```
add the following line:
```yaml
data:
  log_format: $remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent
    $request_time "$http_referer" "$http_user_agent" "$http_x_forwarded_for"
```
##### 3. get the ip adress of the ingress gateway
```
IP=$(kubectl get svc ngninx-nginx-ingress -ojson | jq  '.status.loadBalancer.ingress[].ip')
```
##### 4. update the deployment file
```
sed -i "s,IP_TO_REPLACE,$IP," hipstershop/k8s-manifest.yaml
sed -i "s,IP_TO_REPLACE,$IP," grafana/ingress.yaml
```
### 5. Deploy Prometheus
#### HipsterShop
```
cd hipstershop
kubectl create ns hipster-shop
kubectl -n hipster-shop create rolebinding default-view --clusterrole=view --serviceaccount=hipster-shop:default
kubectl -n hipster-shop apply -f k8s-manifest.yaml
```
#### Prometheus 
```
helm install prometheus stable/prometheus-operator
```
#### Expose Grafana
```
kubectl get svc
kubectl edit svc prometheus-grafana
```
change to type NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  labels:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 7.0.3
    helm.sh/chart: grafana-5.3.0
  name: prometheus-grafana
  namespace: default
  resourceVersion: "89873265"
  selfLink: /api/v1/namespaces/default/services/prometheus-grafana
spec:
  clusterIP: IPADRESSS
  externalTrafficPolicy: Cluster
  ports:
  - name: service
    nodePort: 30806
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
Deploy the ingress by making sure to replace the service name of your grafana
```
cd ..\grafana
kubectl apply -f ingress.yaml
```
Get the login user and password of Grafana
* For the password :
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
* For the login user:
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-user}" | base64 --decode
```

#### Create the ServiceMonitor to scrape the nginx metrics
In order to let the Prometheus scrape the nginx metrics we need to create a ServiceMonitor
```
kubectl apply -f prometheus/servicemonitor.yaml 
```
#### Install Loki with Promtail
```
helm repo add loki https://grafana.github.io/loki/charts
helm repo update
helm upgrade --install loki loki/loki-stack
```
#### Configure Grafana 
In order to build a dashboard with data stored in Loki,we first need to add a new DataSource.
In grafana, goto Configuration/Add data source.
<p align="center"><img src="/image/addsource.PNG" width="60%" alt="grafana add datasource" /></p>
Select the source Loki , and configure the url to interact with it.

Remember Grafana is hosted in the same namesapce as Loki.
So you can simply refer the loki service :
<p align="center"><img src="/image/datasource.PNG" width="60%" alt="grafana add datasource" /></p>

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