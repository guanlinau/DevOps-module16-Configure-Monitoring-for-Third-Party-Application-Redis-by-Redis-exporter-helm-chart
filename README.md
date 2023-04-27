### Project name:

Configure Monitoring for a Third-Party Application

### Technologies used:

Prometheus, Kubernetes, Redis, Helm, Grafana

### Project Description:

1- Setup EKS cluster using eksctl

2- Deploy microservices to k8s cluster

3- Deploy Prometheus, Alert Manager and Grafana in cluster as part of the Prometheus Operator using Helm chart

4-Configure our Monitoring Stack to notify us whenever CPU usage > 50% or Pod cannot start

a-Configure Alert Rules in Prometheus Server

b-Configure Alertmanager with Email Receiver

5-Monitor Redis by using Prometheus Exporter

a-Deploy Redis exporter using Helm Chart

6-Configure Alert Rules (when Redis is down or has too many connections)

7-Import Grafana Dashboard for Redis to visualize monitoring data in Grafana

### Instruction

###### Step 1: Create eksctl in default region

```
aws configure
```

```
eksctl create cluster -name my-cluster
```

# eksctl will configure the credentials with eks cluster for kubectl and save it to kubeconfig

![image](images/Screenshot%202023-04-25%20at%209.16.33%20am.png)
![image](images/Screenshot%202023-04-25%20at%209.21.19%20am.png)

###### Step 2: Deploy microservices to my-cluster

```
kubectl apply -f config-microservices.yaml
```

![image](images/Screenshot%202023-04-25%20at%209.19.21%20am.png)

###### Step 3: Deploy prometheus, alert manager and Grafana via helm chart

#Add repo

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

#Update the repo chart

```
helm repo update
```

#Create a new namespace for prometheus

```
kubectl create namespace monitoring
```

#Deploy prometheus via helm chart

```
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
```

![image](images/Screenshot%202023-04-25%20at%209.27.39%20am.png)

###### Step 4: Forward-port to Prometheus UI

```
kubectl port-forward service/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090 &
```

![image](images/Screenshot%202023-04-25%20at%2011.00.16%20am.png)

###### Step 5: Forward-port to Grafana UI

```
kubectl port-forward service/monitoring-grafana 8080:80 -n monitoring &
```

![image](images/Screenshot%202023-04-25%20at%2011.02.09%20am.png)
![image](images/Screenshot%202023-04-25%20at%2010.19.02%20am.png)

###### Step 6: Simulate a request to microservices front-end to see the Grafana UI

#Create a test pod inside the cluster to send requests to front-end pod

```
kubectl run curl-test --image=radial/busyboxplus:curl -i --ty --rm
```

#Write a test.sh to send 100000 requests to front-end pod endpoint
![image](images/FireShot%20Capture%20028%20-%204%20-%20Introduction%20to%20Grafana%20-%2023_25%20-%2011_11_%20-%20techworld-with-nana.teachable.com.png)

#The output of Grafana
![image](images/Screenshot%202023-04-25%20at%2011.07.13%20am.png)

###### Step 7- Create two new alert rules to prometheus : CPU >50% & Pod cannot start

```
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: main-rules
  namespace: monitoring
  labels:
    app: kube-prometheus-stack
    release: monitoring
spec:
  groups:
    - name: main.rules
      rules:
        - alert: HostHighCPULoad
          expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m]))*100) > 50
          for: 2m
          labels:
            severity: warning
            namespace: monitoring
          annotations:
            description: "CPU load on host is over 50%\n value={{$value}}\n Instance={{$labels.instance}}"
            summary: "Host CPU load high"
        - alert: KubernetesPodCrashLooping
          expr: kube_pod_container_status_restarts_total > 5
          for: 0m
          labels:
            severity: critical
            namespace: monitoring
          annotations:
            description: "Pod {{$labels.pod}} is crash looping\n value={{$value}}"
            summary: "Kubernetes pod crash loop"
```

#Apply the alert-rules.yaml to cluster so prometheus can pick them up

```
kubectl apply -f alert-rules.yaml
```

```
kubectl get PrometheusRule -n monitoring
```

![image](images/Screenshot%202023-04-25%20at%207.49.15%20pm.png)

###### Step 8- Deploy a test-pod into the current cluster to test cpu > 50 alert rule

#Deploy a test-pod

```
kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 30s --metrics-briefy
```

```
kubectl get pod
```

![images](images/Screenshot%202023-04-25%20at%208.28.16%20pm.png)

![images](images/Screenshot%202023-04-25%20at%208.29.50%20pm.png)

###### Step 9- Configure Alert manager

#Create a secret.yaml for alert-manager-config.yaml to login gmail account

```
kubectl apply -f email-secret.yaml
```

#Create the alert-manager-config.yaml

```
kubectl apply -f alert-manager-config.yaml
```

#Check the alert-manager

```
kubectl get alertmanagerconfig -n monitoring
```

![images](images/Screenshot%202023-04-26%20at%2010.59.35%20am.png)
#Test the email notification via Test cpu stress pod
![images](images/Screenshot%202023-04-26%20at%2012.33.08%20pm.png)

###### Step 10 : Monitor Redis by using Prometheus Exporter and install serviceMonitor

#Create redis-values.yaml to overwrite the default values in redis-exporter helm chart
#Enable serviceMonitor, so prometheus will pickup target after deployment

```
---
serviceMonitor:
  enabled: true
  labels:
    release: monitoring

redisAddress: redis://redis-cart:6379

```

#Install redis-exporter via helm chart

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install redis-exporter prometheus-community/prometheus-redis-exporter -f redis-values.yaml
```

![images](images/Screenshot%202023-04-26%20at%208.16.13%20pm.png)
![images](images/Screenshot%202023-04-26%20at%208.34.19%20pm.png)
![images](images/Screenshot%202023-04-26%20at%208.20.55%20pm.png)
![images](images/Screenshot%202023-04-26%20at%208.23.46%20pm.png)

###### Step 11: Configure alert rule for redis

#Using rule template
![images](images/Screenshot%202023-04-27%20at%209.29.47%20am.png)

###### Step 12: Import Grafana dashboard for redis for visualizing monitoring redis data

#Using grafara template
#Import the template in grafara dashboard
![images](images/Screenshot%202023-04-27%20at%209.49.04%20am.png)
