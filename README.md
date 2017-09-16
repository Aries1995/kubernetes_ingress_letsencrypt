# kubernetes_ingress_letsencrypt

![](https://i.imgur.com/bbCz37T.png)
## 申請Domain Name
首先就是申請一個你要的網域, 這邊網路上資源很多都可以查一下哪個網域商或是一些相關的建議,這邊我就先不去多做介紹了,文章中會以sam.nctu.me來作範例



## 用[Letsencrypt](https://letsencrypt.org/)來簽發憑證
這邊我們用手動的把它先簽下來,上[Letsencrypt](https://letsencrypt.org/)去安裝certbot, 手動簽的方式也可以參考我的上一篇[簽letsencrpyt憑證](https://www.nctusam.com/2017/08/30/ubuntumint-nginx-%E7%B0%BDletsencrpyt%E6%86%91%E8%AD%89/) 文章,輸入以下指令來簽憑證


```shell
$ sudo certbot certonly --standalone sam.nctu.me
```
結果會如下方顯示
```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/sam.nctu.me/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/sam.nctu.me/privkey.pem
   Your cert will expire on 2017-12-15. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

```


簽好後會自動放在

```
/etc/letsencrypt/live/sam.nctu.me/
```
可以看到裡面的憑證,權限必須是root才能看到這個資料夾的內容
```shell
# ls
cert.pem  chain.pem  fullchain.pem  privkey.pem  README
```

這樣憑證就簽完了

## 設置Kubernetes Secret

將fullchain.pem(因為使用nginx controller) 與 privkey.pem來建立secret,這之後會配置給ingress使用

```shell
$ kubectl create secret tls tls-certs --cert fullchain.pem --key privkey.pem 
```
查看secret
```shell
$ kubectl get secret
NAME                              TYPE                                  DATA      AGE
default-token-rtlbl               kubernetes.io/service-account-token   3         6d
tls-certs                         kubernetes.io/tls                     2         55m
```
以建立完成


## 建立Ingress
我們先去建立一個ingress的yaml檔,來幫助我們create ingress並加入tls的設定
ingress.yaml
```yaml=
apiVersion: extensions/v1beta1                                                                                
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  tls:
    - hosts:
      - sam.nctu.me
      secretName: tls-certs
  rules:
  - host: sam.nctu.me
    http:
      paths:
        - backend:
            serviceName: phpmyadmin
            servicePort: 80
```
然後建立此Ingress

```shell
$ kubectl create -f ingress.yaml
```
並查看
```shell
$ kubectl get ingress 
NAME            HOSTS            ADDRESS   PORTS     AGE
nginx-ingress   sam.nctu.me             80, 443   1h
```

## 建立default-backend
先建立一個delfault,如果試著用ip直接request或是沒有設定過的domain name,會回傳404

default-backend.yaml
```yaml=
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    k8s-app: default-http-backend
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        # Any image is permissable as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: gcr.io/google_containers/defaultbackend:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: kube-system
  labels:
    k8s-app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    k8s-app: default-http-backend
```

```shell
$ kubectl create -f default-backend.yaml
```
## 建立Ingress-Nginx-Controller
這邊有個小坑,在建立之前要先設置RBAC才會順利建起來,花了點時間才發現,所以我們要先建立相關的RBAC
ingress-controller-rbac.yaml
```yaml=
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: ingress
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - services
  - endpoints
  - ingresses
  - nodes
  - pods
  verbs:
  - list
  - watch
- apiGroups:
  - "extensions"
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - events
  - services
  verbs:
  - create
  - list
  - update
  - get
  - patch
- apiGroups:
  - "extensions"
  resources:
  - ingresses/status
  - ingresses
  verbs:
  - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: ingress-ns
  namespace: kube-system
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - get
  - create
  - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: ingress-ns-binding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-ns
subjects:
  - kind: ServiceAccount
    name: ingress
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: ingress-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress
subjects:
  - kind: ServiceAccount
    name: ingress
    namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress
  namespace: kube-system
```

```shell
$ kubectl create -f ingress-controller-rbac.yaml
```

之後就可以建立Ingress-controller
nginx-ingress-daemonset.yaml
```yaml=
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  labels:
    k8s-app: nginx-ingress-controller
  namespace: kube-system
spec:
  template:
    metadata:
      labels:
        k8s-app: nginx-ingress-controller
    spec:
      # hostNetwork makes it possible to use ipv6 and to preserve the source IP correctly regardless of docker configuration
      # however, it is not a hard dependency of the nginx-ingress-controller itself and it may cause issues if port 10254 already is taken on the host
      # that said, since hostPort is broken on CNI (https://github.com/kubernetes/kubernetes/issues/31307) we have to use hostNetwork where CNI is used
      # like with kubeadm
      hostNetwork: true
      terminationGracePeriodSeconds: 60
      serviceAccountName: ingress
      containers:
      - image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.3
        name: nginx-ingress-controller
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          timeoutSeconds: 1
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        args:
        - /nginx-ingress-controller
        - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
```
```shell
$ kubectl create -f nginx-ingress-daemonset.yaml
```
查看是否有建立完成
```shell
$ kubectl get pod -n kube-system
nginx-ingress-controller-3q2b0           1/1       Running   0          1h
nginx-ingress-controller-j28sz           1/1       Running   0          1h
nginx-ingress-controller-lrn34           1/1       Running   0          1h

```
然後再測試 直接輸入 https://sam.nctu.tw 就行了, 當然這邊只是測試,記得換成你設定的domain name

