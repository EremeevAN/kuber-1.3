### Домашнее задание к занятию «Запуск приложений в K8S»
### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

Создаем Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginxmultitool
  annotations:
    container1: nginx
    container2: multitools
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - name: web
              containerPort: 80
          livenessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 10
            timeoutSeconds: 3
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 4
        - name: multitool
          image: praqma/network-multitool:alpine-extra
          resources:
          env:
            - name: HTTP_PORT
              value: "1181"
            - name: HTTPS_PORT
              value: "1443"
          ports:
            - name: http
              containerPort: 1181
              protocol: TCP
            - name: https
              containerPort: 1443
              protocol: TCP
          livenessProbe:
            tcpSocket:
              port: 1181
            initialDelaySeconds: 10
            timeoutSeconds: 3
          readinessProbe:
            httpGet:
              path: /
              port: 1181
            initialDelaySeconds: 15
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 2
```
В результате у нас запускается один под:

![images](https://github.com/EremeevAN/kuber-1.3/blob/main/images/1.png)
поменяем в строке   replicas: 1 на   replicas: 2

Получаем на выходе 2 пода:

![images](https://github.com/EremeevAN/kuber-1.3/blob/main/images/2.png)
Создаем сервис и под для доступа до приложений

```yaml
apiVersion: v1
kind: Service
metadata:
  name: one
spec:
  selector:
    app: nginx-multitool
  type: ClusterIP
  ports:
    - name: nginx
      port: 8080
      targetPort: 80
    - name: multitool-http
      port: 8081
      targetPort: 1181
    - name: multitool-https
      port: 8443
      targetPort: 1443
...
apiVersion: v1
kind: Pod
metadata:
  name: pod-multitool
  labels:
    app: multitool
spec:
  containers:
    - name: multitool
      image: praqma/network-multitool:alpine-extra
      resources:
      ports:
        - name: "http"
          containerPort: 8082
        - name: "https"
          containerPort: 8444
```

Получаем результат:

![images](https://github.com/EremeevAN/kuber-1.3/blob/main/images/3.png)

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

Создаем Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  annotations:
    container: nginx
    container-init: busybox
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: nginx-init
  template:
    metadata:
      labels:
        app: nginx-init
    spec:
      containers:
        - name: nginx
          image: nginx:1.24.0
          ports:
            - name: web
              containerPort: 80
          livenessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 10
            timeoutSeconds: 3
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 4
      initContainers:
        - name: busybox
          image: busybox:1.36.1
          resources:
          env:
            - name: TARGET
              value: "two"
          command: ['sh', '-c', "until nslookup $TARGET.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do sleep 2; done"]
```

![images](https://github.com/EremeevAN/kuber-1.3/blob/main/images/4.png)

Видим что init-pod не хочет стартовать. Создаем сервис

```yaml
apiVersion: v1
kind: Service
metadata:
  name: two
spec:
  selector:
    app: nginx-init
  type: ClusterIP
  ports:
    - name: nginx
      port: 8888
      targetPort: 80
```

Видим что Деплоймент заработал

![images](https://github.com/EremeevAN/kuber-1.3/blob/main/images/5.png)