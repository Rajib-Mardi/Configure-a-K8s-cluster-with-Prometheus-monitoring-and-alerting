### Monitoring with Prometheus

<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/0a39c835-faa8-433c-90d5-429c503c0b7d" width="70">         <img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/bfeafcb7-7498-4d1c-816c-a7902c9359c4" width="295"> 

__________________________________________________________________

### Project: 
 * Install Prometheus Stack in Kubernetes
### Technologiesused: 
  * Prometheus, Kubernetes, Helm, AWS EKS, eksctl, Grafana, Linux
#### Project Description:

* Using the eksctl command line, create an EKS cluster with the default configuration, region, and two worker nodes.
  
* Deploy microservices Application

  * Deploy the online Shop microservices Application in the cluster with the Yaml file configuration
  ```
  kubectl apply -f shopping-cart-config.yaml
  ```

##### Deploy Prometheus, Alert Manager and Grafana in cluster as part of the Prometheus Operator using Helm chart

* Add the helm repository for prometheus Operator

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update
```

 * Install the prometheus into the own namespace
 * Create the namespace
```
kubectl create namespace monitoring
```
* Now , Install the prometheus Chart in monitoring Namespace

```
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
```


<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/65441565-6aa9-4a9b-be4f-c6b9800e941b" width="750">

* Access the prometheus UI
```
kubectl port-forward service/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090 &
```

* For  proper Visualization Access the Grafana UI

```
kubectl port-forward service/monitoring-grafana -n monitoring 8080:80
```
* Login the Credentials in Grafana UI using User as ```admin ``` and password as ```prom-operator```


* Test CPU Spike in the cluster  in the grafana UI 
* Test the CPU spike for total load on the CPU in the cluster.
* so test the busybox for the CPU spike 
```
kubectl run curl-test --image=radial/busyboxplus:curl -i --tty --rm
```

* Inside the terminal pod , execute the script that will hit the endpoint of the application  1000 or 10000 times
* create the .sh file
```
vi test.sh
```


<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/64232415-e614-42f6-82f0-cbc69dba8ecd" width="750">

* make it executable and  execute  the ```./test.sh```

* In grafana Graph we see the CPU spike in  the cluster


<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/44f520f0-715f-4adc-a910-375c36bef321" width="750">



----------------------------------------------------------------------------


### Project: 
  * Configure Alerting for our Application
### Technologiesused: 
  * Prometheus, Kubernetes, Linux

#### Project Description:

#### Configure our Monitoring Stack to notify us whenever CPU usage > 50% or Pod cannot start

 #### Configure Alert Rules in Prometheus Server

* Create a alert rule yaml file and apply it
#### YAML code defines a PrometheusRule named "main-rules" in the "monitoring" namespace. The rule contains two alerting rules specified under the "main.rules" group.
1. ```Alert: HostHighCpuLoad```

  * ```Expression (expr)```:
     ```
     100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 50
     ```

* This alert triggers when the CPU load on the host is over 50%. It calculates the average CPU idle time over the last 2 minutes and alerts if the result is less than 50%.
 * ```For```:
    * Alerts if the condition is true for 2 minutes.
 * ```Labels```:
   * ```severity: warning```
   * ```namespace: monitoring```
 * ```Annotations```:
   * ```description:``` provides details about the alert, including the CPU load percentage and the instance.
   * ```summary:``` gives a brief summary of the alert.

2. ```Alert: KubernetesPodCrashLooping```

  * ```Expression (expr)```:
    ```
    kube_pod_container_status_restarts_total > 5
    ```
* This alert triggers if the total number of restarts for a pod's containers is greater than 5.
  * ```For```:
    * Alerts immediately (for: 0m).
  * ```Labels:```  
    * ```severity: critical```
    * ```namespace: monitoring```
  * ```Annotations:```
     * ```description```: provides details about the pod that is crash-looping and the number of restarts.
     * ```summary:``` gives a brief summary of the alert.

```
 - Ref - alert-rules.yaml
 - After Creating rule apply the file = kubectl apply -f <alert-rule file>
```




<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/d6809e15-aaf0-4f48-bd87-687615777b94" width="750">

* Test Our Alert Rules
* We goona simulate a CPU load in our cluster to trigger the alert 

```
  Using CPUStress docker image we will run the container inside the k8s pod for testing our rule.
  - `kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 30s --metrics-brief`
  You can check the cpu load over grafana under dashboard 'kubernetes-cluster'
  Also, Can check on prometheus alert.
```


<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/3761c56a-a0b4-45c2-b06f-c27365b16275" width="750">

#### Configure Alertmanager with Email Receiver
##### YAML code defines an AlertmanagerConfig named "main-rules-alert-config" in the "monitoring" namespace. This configuration specifies how alerts should be routed and how notifications should be sent


1. ```Routing Configuration (spec.route):```

  * ```Default Receiver (receiver):```
     * All alerts are routed to the 'email' receiver by default.
  * ```Default Repeat Interval (repeatInterval):```
     * 30 minutes between repeating notifications for the same alert.
   * ```Routes (routes):```
     * Two routes are defined, each matching a specific alert name.
        1.1 For alerts with the name "HostHighCpuLoad":
        * No additional configuration, so it uses the default receiver and repeat interval.
        1.2 For alerts with the name "KubernetesPodCrashLooping":
         * Custom repeat interval of 10 minutes.
2. ```Receiver Configuration (spec.receivers):```

   * ```Receiver Definition (receivers):```
      * A single receiver named 'email' is defined.
   * ```Email Configuration (emailConfigs):```
   * Email configurations for sending notifications via email.
     * ```to```: Email address to send notifications to.
     * ```from```: Email address from which notifications will be sent.
     * ```smarthost```: The SMTP server's address and port.
     * ```authUsername```: Username for authentication.
     * ```authIdentity```: Identity for authentication (typically the same as the username).
     * ```authPassword```: Password for authentication, which is stored in a secret named "gmail-auth" with the key "password".
    
   * Configured the email-secret.yaml  for email password to be kept as secret 

```
To Check Prometheus Alert UI -
    - kubectl port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093 -n monitoring &
 Create an alert-manager.yaml file
 Create  secret-email.yaml file 
    - Ref - alert-manager.yaml
    - ref - secret-email.yaml
  Apply the file
    - kubectl apply -f email-secret.yaml
    - kubectl apply -f alert-manager.yaml

```

#### Test the Email Notification
  *  Test Our own alert manager

```
Load the CPUStress by using CPUStress Image, and when HostHighCpuLoad alert rule is in firing state, you will get email.
- `kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 60s --metrics-brief`
```




<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/449d2bf1-59e8-43bc-af97-99011ef2f7db" width="750">



<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/db05fb48-ba75-4aec-a9e2-cc16a603da7d" width="750">

####  Check the email  for alert firing send the notication to the email



<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/85edcabc-1408-4c13-bc96-3b925b0b5f0a" width="750">


<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/cda792a4-a06b-4306-8e76-ea7ee8322bb1" width="750">



---------------------------------------------------------------------------------


### Project: 
  * Configure Monitoring for a Third-Party Application(Redis)
 ### Technologiesused: Prometheus, Kubernetes, Redis, Helm, Grafana
 ### Project Description:

 * Monitor Redis by using Prometheus Exporter

 * Deploy Redis service in our cluster

 *  Deploy Redis exporter using Helm Chart

```
  Using Helm Chart to deploy
    - helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    - helm repo update
    - helm install redis-exporter prometheus-community/prometheus-redis-exporter  redis-values.yaml

```


<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/56d474dd-a11e-4c40-adfe-f10efcfe5c88" width="750">

 Check the redis exporter target
 ```
    - Go to Prometheus UI
    - Click on Status => Target
    - Search for redis-exporter
```

<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/c097b400-28b2-4e75-9e88-41538f60df2b" width="750">

##### Configure Alert Rules (when Redis is down or has too many connections)


#####  YAML code defines a PrometheusRule named "redis-rule" with two alerting rules in the "redis.rules" group. 

1. ```Alert: RedisDown```

   * Expression (expr):
   ```
    redis_up == 0
   ```
    * This alert triggers when the redis_up metric is equal to 0, indicating that the Redis instance is down.
  * ```For:```
    * Alerts immediately (for: 0m).
  * ```Labels:```
    * ```severity: critical```
  * ```Annotations:```
    * ```summary```: provides a brief summary of the alert, indicating that Redis is down for the specified instance.
    * ```description```: gives more details, including the metric value and labels.
2. ```Alert: RedisTooManyConnections```

  * Expression (expr):
    ```
     redis_connected_clients / redis_config_maxclients * 100 > 90
    ```
  * This alert triggers when the percentage of connected clients exceeds 90% of the maximum configured clients.
  * ```For:```
   * Alerts if the condition is true for 2 minutes.
  * ```Labels:```
  * ```severity: warning```
  * ```Annotations:```
     * ```summary:``` provides a brief summary of the alert, indicating that Redis has too many connections for the specified instance.
    * ```description:``` gives more details, including the metric value and labels.

#### Create Alert Rules for Redis
```
 - Ref - redis-rules.yaml
  - After Creating rule apply the file = kubectl apply -f <alert-rule file>
```
#### check your redis rule in prometheus UI
```
  - Go to Prometheus UI
  - Click on Alert
  - Search for redis-rules
```

<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/bd34c6a4-bd99-4971-a5d8-280d03c2f76e" width="750">

#### Test the redis metrics

* edit the redis-cart deployment 
  * set the no. of replicas to zero
    
   ```
   kubectl edit deployment redis-cart
   ```
* After that redis-cart will be terminating and redis metrics will be not available

<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/84d10f4f-dc3c-4af5-a0a5-ed047abe1d14" width="750">


#### Create Redis Dashboard in  Grafana
* Import Grafana Dashboard for Redis to visualize monitoring data in Grafana
 ```
  Search Predefined Dashboard on grafana labs(https://grafana.com/grafana/dashboards)
    - Predefined Dashboard should have to use the same exporter.
    - For Redis Predefined Dashboard - https://grafana.com/grafana/dashboards/763-redis-dashboard-for-prometheus-redis-exporter-1-x/
```


<img src="https://github.com/Rajib-Mardi/Monitoring-with-Prometheus/assets/96679708/7cd9e141-99fc-44c5-981c-efbea3ad98b9" width="750">
