# Node Agent Logging Service
## 01 設定elasticSearch 
- https://hackmd.io/@ShannonHung/BJX8R1xnc
```
$ kubectl create -f es-ns.yaml
$ kubectl create -f es-svc.yaml
$ kubectl create -f es-cluster.yaml
```
## 02 建立Kibana 
```
$ kubectl create -f kibana.yaml
```
## 03-1 設定fluntd 的簡單作法沒有config
- https://devopscube.com/setup-efk-stack-on-kubernetes/

```
# 建立cluster role可以 授權pods, ns的權限
$ kubectl create -f fluentd-role.yaml

# 建立service account這個sa會被pod使用 
$ kubectl create -f fluentd-sa.yaml

# cluter role binding 把定義在cluster-role的permission bind到service account
$ kubectl create -f fluentd-rb.yaml

# DaemonSet建立確保每個node都有一個fluentd pod
$ kubectl create -f fluentd-ds.yaml
```
## 03-2 設定fluentd 困難的方法 有config設置
- 參考網站: https://www.qikqiak.com/post/install-efk-stack-on-k8s/
- 還有可以設定相關label在pod上才會收log的機制
- 為了能夠靈活控制那些node的log可以收集所以我們有添加nodeSelector屬性
- 怎麼樣的pod或是node才會被fluentd收集相關的log? 


```
# 先建立相關 fluentd.config
$ kubectl create -f fluentd-configmap.yaml

# 建立cluster role, service account, clusterRoleBinding, DaemonSet 
$ kubectl create -f fluentd-daemonset.yaml
```


1. node有添加以下標籤 `beta.kubernetes.io/fluentd-ds-ready: "true"`
```
# 請使用以下指令確認node有以下標籤
$ kubectl get nodes --show-labels

# fluentd DaemonSet的設置
...
nodeSelector:
  beta.kubernetes.io/fluentd-ds-ready: "true"

# 對node打label
$ kubectl label nodes <node-name> beta.kubernetes.io/fluentd-ds-ready=true

... 
```
2. pod有添加以下標籤只保留具有`logging=true`标签的Pod日志
```
# 請使用以下指令確認pod有以下標籤
$ kubectl get pod --show-labels
...
metadata:
  name: counter
  labels:
    logging: "true"  # 一定要具有该标签才会被采集
```
# SideCar Logging Service 
我們現在要做的事情是在pod裡面建立兩個container分別是app以及fluentd。
```bash
# 建立fluentd的configMap檔案 
$ kubectl create -f kioxia-fluentd-configMap.yaml
# 把app跑起來create deployment 會同時建立兩個container
$ kubectl create -f kioxia-deployment.yaml
```
## 驗證
因為我們服務的log是產生在fluentd的/logs底下，以及kioxia-pep的/logs底下
所以我們透過fluentd tail的source來不斷的讀取相對應的檔案
並且送去elasticSearch 
這邊要注意的是，記得去kioxia-fluentd-configMap.yaml修改host
**注意: 因為es是建立在namespace為logging的svc，因此host要修改成es-cluster-svc.logging.svc.cluster.local**

> 應該要看到以下兩種檔案確定mount有對
```
$ kubectl exec -it -n kioxia kioxia-pep-<tab> fluentd-es -- sh 
# cd /logs 
accessLog.log

# cd /etc/fluentd/confit.d
output.d
```