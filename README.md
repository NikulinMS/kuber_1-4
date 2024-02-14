# Домашнее задание к занятию "`Сетевое взаимодействие в K8S. Часть 1`" - `Никулин Михаил Сергеевич`



---

## Задание 1. Создать Deployment и обеспечить доступ к контейнерам приложения по разным портам из другого Pod внутри кластера

### 1. Создать Deployment приложения, состоящего из двух контейнеров (nginx и multitool), с количеством реплик 3 шт.
Создадим пространство имен для ДЗ:
```bash
nikulinn@nikulin:~/other/kuber_1-4$ kubectl create namespace dz4
namespace/dz4 created
nikulinn@nikulin:~/other/kuber_1-4$ kubectl get ns
NAME              STATUS   AGE
kube-system       Active   2d
kube-public       Active   2d
kube-node-lease   Active   2d
default           Active   2d
dz4               Active   6s
```
Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment
  namespace: dz4
  labels:
    app: main
spec:
  replicas: 3
  selector:
    matchLabels:
      app: main
  template:
    metadata:
      labels:
        app: main
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.2
      - name: multitool
        image: wbitt/network-multitool
        env:
          - name: HTTP_PORT
            value: "8080"
```
```bash
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl apply -f my_deployment.yaml 
deployment.apps/netology-deployment created
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl get pods -n dz4 -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
netology-deployment-77497ddc64-8lgh9   2/2     Running   0          26s   10.1.123.162   netology-01   <none>           <none>
netology-deployment-77497ddc64-4t9gd   2/2     Running   0          26s   10.1.123.163   netology-01   <none>           <none>
netology-deployment-77497ddc64-rmjhc   2/2     Running   0          26s   10.1.123.164   netology-01   <none>           <none>
```
### 2. Создать Service, который обеспечит доступ внутри кластера до контейнеров приложения из п.1 по порту 9001 — nginx 80, по 9002 — multitool 8080.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: deployment-service
  namespace: dz4
spec:
  ports:
    - name: http-app
      port:  9001
      protocol: TCP
      targetPort: 80
    - name: http-app-mult
      port:  9002
      protocol: TCP
      targetPort: 8080
  selector:
    app: main
```
```bash
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl apply -f svc_nginx.yaml 
service/deployment-service created
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl get svc -n dz4 -o wide
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE   SELECTOR
deployment-service   ClusterIP   10.152.183.134   <none>        9001/TCP,9002/TCP   16s   app=main
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl get ep -n dz4 -o wide
NAME                 ENDPOINTS                                                           AGE
deployment-service   10.1.123.162:8080,10.1.123.163:8080,10.1.123.164:8080 + 3 more...   3m7s

```
### 3. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложения из п.1 по разным портам в разные контейнеры.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-multitool
  namespace: dz4
spec:
  containers:
    - image: wbitt/network-multitool
      name: test-multitool
```
```bash
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl apply -f test_multitool.yaml 
pod/test-multitool created
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl get pods -n dz4 -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
netology-deployment-77497ddc64-8lgh9   2/2     Running   0          5m51s   10.1.123.162   netology-01   <none>           <none>
netology-deployment-77497ddc64-4t9gd   2/2     Running   0          5m51s   10.1.123.163   netology-01   <none>           <none>
netology-deployment-77497ddc64-rmjhc   2/2     Running   0          5m51s   10.1.123.164   netology-01   <none>           <none>
test-multitool                         1/1     Running   0          7s      10.1.123.165   netology-01   <none>           <none>
```
Тестируем, что ПОДы отыечают по ожидаемым портам:
```bash
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl 10.1.123.162:80
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0  36745      0 --:--:-- --:--:-- --:--:-- 38250
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl 10.1.123.162:8080
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   155  100   155    0     0  96814      0 --:--:-- --:--:-- --:--:--  151k
WBITT Network MultiTool (with NGINX) - netology-deployment-77497ddc64-8lgh9 - 10.1.123.162 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl 10.1.123.163:80
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   432k      0 --:--:-- --:--:-- --:--:--  597k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl 10.1.123.163:8080
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   155  100   155    0     0   105k      0 --:--:-- --:--:-- --:--:--  151k
WBITT Network MultiTool (with NGINX) - netology-deployment-77497ddc64-4t9gd - 10.1.123.163 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl 10.1.123.164:80
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   501k      0 --:--:-- --:--:-- --:--:--  597k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl 10.1.123.164:8080
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   155  100   155    0     0  14693      0 --:--:-- --:--:-- --:--:-- 15500
WBITT Network MultiTool (with NGINX) - netology-deployment-77497ddc64-rmjhc - 10.1.123.164 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
```
Тестируем, что Сервис отвечает по нужным портам:
```bash
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl 10.152.183.134:9001
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   526k      0 --:--:-- --:--:-- --:--:--  597k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl 10.152.183.134:9002
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   155  100   155    0     0   152k      0 --:--:-- --:--:-- --:--:--  151k
WBITT Network MultiTool (with NGINX) - netology-deployment-77497ddc64-rmjhc - 10.1.123.164 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
```
### 4. Продемонстрировать доступ с помощью `curl` по доменному имени сервиса.
```bash
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl deployment-service:9001
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   196k      0 --:--:-- --:--:-- --:--:--  298k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl exec -n dz4 test-multitool -- curl deployment-service:9002
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   155  100   155    0     0  75646      0 --:--:-- --:--:-- --:--:--  151k
WBITT Network MultiTool (with NGINX) - netology-deployment-77497ddc64-rmjhc - 10.1.123.164 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
```
### 5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4.

------

## Задание 2. Создать Service и обеспечить доступ к приложениям снаружи кластера

### 1. Создать отдельный Service приложения из Задания 1 с возможностью доступа снаружи кластера к nginx, используя тип NodePort.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: deployment-service-nodeport
  namespace: dz4
spec:
  ports:
    - name: http-app
      port:  80
      nodePort: 30000
    - name: http-app-mult
      port:  8080
      nodePort: 30001
  selector:
    app: main
  type: NodePort
```
```bash
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl apply -f svc_nginx_nodeport.yaml 
service/deployment-service-nodeport created
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl get svc -n dz4 -o wide
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE   SELECTOR
deployment-service            ClusterIP   10.152.183.134   <none>        9001/TCP,9002/TCP             19m   app=main
deployment-service-nodeport   NodePort    10.152.183.44    <none>        80:30000/TCP,8080:30001/TCP   29s   app=main
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl get ep -n dz4 -o wide
NAME                          ENDPOINTS                                                           AGE
deployment-service            10.1.123.162:8080,10.1.123.163:8080,10.1.123.164:8080 + 3 more...   20m
deployment-service-nodeport   10.1.123.162:8080,10.1.123.163:8080,10.1.123.164:8080 + 3 more...   45s
nikulinn@nikulin:~/other/kuber_1-4/scr$ kubectl get nodes -o wide
NAME          STATUS   ROLES    AGE    VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION    CONTAINER-RUNTIME
netology-01   Ready    <none>   2d1h   v1.28.3   192.168.101.26   <none>        Debian GNU/Linux 11 (bullseye)   5.10.0-19-amd64   containerd://1.6.15
```
### 2. Продемонстрировать доступ с помощью браузера или `curl` с локального компьютера.
```bash
2. nikulinn@nikulin:~/other/kuber_1-4/scr$ curl 178.154.200.234:30000
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
nikulinn@nikulin:~/other/kuber_1-4/scr$ curl 178.154.200.234:30001
WBITT Network MultiTool (with NGINX) - netology-deployment-77497ddc64-4t9gd - 10.1.123.163 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
```
![imgt_1.png](img%2Fimgt_1.png)
![imgt_2.png](img%2Fimgt_2.png)
### 3. Предоставить манифест и Service в решении, а также скриншоты или вывод команды п.2.

------
