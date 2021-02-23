# 腾讯云k8s 1.19.0+Traefik 2.0.5部署
### 一、Traefik简介
 Traefik 最新推出了 v2.0 版本，这里将尝试升级到最新，简单的介绍了下如何在 Kubernetes 环境下安装 Traefik v2.0，在 Traefik v2.0 版本后，配置 Ingress 路由规则其使用了自定义 CRD 对象来完成，并不像之前 1.0+ 版本使用 Kubernetes 自带的 Ingress 对象加注解方式来完成路由配置，下面将介绍如何在 Kubernetes 环境下部署并配置 Traefik v2.0。
### 二、Kubernetes 部署 Traefik

 注意：这里 Traefik 是部署在 Kube-system Namespace 下，如果不是需要修改下面部署文件中的 Namespace 属性。 

##### 1、创建 CRD 资源
```bash
## IngressRoute
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
---
## IngressRouteTCP
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
---
## Middleware
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption

执行该文件：
$ kubectl apply -f traefik-crd.yaml -n kube-system
```
##### 2、创建 RBAC 权限

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kube-system
  name: traefik-ingress-controller
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","secrets"]
    verbs: ["get","list","watch"]
  - apiGroups: ["extensions"]
    resources: ["ingresses"]
    verbs: ["get","list","watch"]
  - apiGroups: ["extensions"]
    resources: ["ingresses/status"]
    verbs: ["update"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["middlewares"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["ingressroutes"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["ingressroutetcps"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["tlsoptions"]
    verbs: ["get","list","watch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: kube-system

执行该文件：
$ kubectl apply -f traefik-rbac.yaml -n kube-system
```

##### 3、创建 Traefik 配置文件

```bash
kind: ConfigMap
apiVersion: v1
metadata:
  name: traefik-config
data:
  traefik.yaml: |-
    serversTransport:
      insecureSkipVerify: true
    api:
      insecure: true
      dashboard: true
      debug: true
    metrics:
      prometheus: ""
    entryPoints:
      web:
        address: ":80"
      websecure:
        address: ":443"
    providers:
      kubernetesCRD: ""
    log:
      filePath: ""
      level: error
      format: json
    accessLog:
      filePath: ""
      format: json
      bufferingSize: 0
      filters:
        retryAttempts: true
        minDuration: 20
      fields:
        defaultMode: keep
        names:
          ClientUsername: drop
        headers:
          defaultMode: keep
          names:
            User-Agent: redact
            Authorization: drop
            Content-Type: keep
            
执行该文件：
$ kubectl apply -f traefik-config.yaml -n kube-system
```

##### 4、Kubernetes 部署 Traefik

```bash
apiVersion: v1
kind: Service
metadata:
  name: traefik
spec:
  ports:
    - name: web
      port: 80
    - name: websecure
      port: 443
    - name: admin
      port: 8080
  selector:
    app: traefik
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: traefik-ingress-controller
  labels:
    app: traefik
spec:
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      name: traefik
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 1
      containers:
        - image: traefik:v2.0.5
          name: traefik-ingress-lb
          ports:
            - name: web
              containerPort: 80
              hostPort: 80           #hostPort方式，将端口暴露到集群节点
            - name: websecure
              containerPort: 443
              hostPort: 443          #hostPort方式，将端口暴露到集群节点
            - name: admin
              containerPort: 8080
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 1000m
              memory: 1024Mi
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
          args:
            - --configfile=/config/traefik.yaml
          volumeMounts:
            - mountPath: "/config"
              name: "config"
      volumes:
        - name: config
          configMap:
            name: traefik-config 
      tolerations:              #设置容忍所有污点，防止节点被设置污点
        - operator: "Exists"
      nodeSelector:             #设置node筛选器，在特定label的节点上启动
        IngressProxy: "true"
        
 执行该文件：
 $ kubectl apply -f traefik-deploy.yaml -n kube-system
```

### 三、Traefik 路由规则配置

##### 1、配置 HTTP 路由规则

* 创建 Traefik Tomcat 路由规则文件 Tomcat-route.yaml

```bash
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-Tomcat-route
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`ts.123.top`)
      kind: Rule
      services:
        - name: tomcat
          port: 8080
```

* 创建 Traefik Dashboard 路由规则对象

```bash
$ kubectl apply -f traefik-dashboard-route.yaml -n kube-system
```

##### 2、配置 HTTPS路由规则

* 将申请的证书上传到master主机上，把证书目录重命名

```bash
cp xxxxx.crt tls.crt
cp xxxxx.key tls.key
或者
mv xxxxx.crt tls.crt
mv xxxxx.key tls.key
```

*  创建证书文件 

```bash
$ kubectl create secret generic cloud-mydlq-tls2 --from-file=tls.crt --from-file=tls.key -n kube-system
```

*  创建 Traefik Nginx 路由规则文件 Nginx-route.yaml

```bash
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-Nginx-route
spec:
  entryPoints:
    - websecure
  tls:
    secretName: cloud-mydlq-tls2
  routes:
    - match: Host(`ts.123.top`)
      kind: Rule
      services:
        - name: nginx
          port: 80
 
#注：我这里选择的入口是websecure，tls选择的是cloud-mydlq-tls2，我的域名是ts.123.top，转入后端服务为Nginx。
```

* 创建
```bash
$ kubectl apply -f traefik-dashboard-route.yaml -n kube-system
```
