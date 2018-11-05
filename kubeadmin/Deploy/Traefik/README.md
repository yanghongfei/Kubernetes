### 12 部署Traefik Ingress

> Traefik 使用DS方式进行部署

```yaml
$ cat <<EOF > traefik-rbac.yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
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
  namespace: default
EOF
```

```yaml
$ cat <<EOF > traefik-ui.yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: default
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: k8s-traefik.shinezone.com
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
EOF
```

```yaml
$ cat <<EOF > traefik-ds.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: default
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: default
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      restartPolicy: Always
      volumes:
      - name: ssl
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-conf
      containers:
      - image: traefik
        name: traefik-ingress-lb
        volumeMounts:
        - mountPath: "/etc/kubernetes/ssl"  #证书所在目录
          name: "ssl"
        - mountPath: "/root/K8S-Online/Traefik"        #traefik.toml文件所在目录
          name: "config"
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8080
        - name: https
          containerPort: 443
          hostPort: 443
        securityContext:
          privileged: true
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --configfile=/root/K8S-Online/Traefik/traefik.toml
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: default
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: http
    - protocol: TCP
      port: 443
      name: https
    - protocol: TCP
      port: 8080
      name: admin
  type: NodePort
EOF
```

#### 12.1 配置Tracefik HTTPS

```yaml
$ cat <<EOF > traefik.toml
defaultEntryPoints = ["http", "https"]

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      certFile = "/root/ssl/shinezone.com.crt"
      keyFile = "/root/ssl/shinezone.com.key"
EOF
```

```shell
$ kubectl create configmap traefik-conf --from-file=traefik.toml    //生成配置字典
$ kubectl create secret generic traefik-cert --from-file=/etc/kubernetes/ssl/shinezone.com.key --from-file=/etc/kubernetes/ssl/shinezone.com.crt                    //生成保密字典
```

#### 12.2 部署Traefik

```shell
$ kubectl create -f traefik-rbac.yaml 
$ kubectl create -f traefik-ds.yaml
$ kubectl create -f traefik-ui.yaml
```

#### 12.3 访问Traefik

> 默认端口Node IP+ NodePort 8080 ,也可以使用Traefik代理出来 使用域名访问

http://172.16.1.10:8080/dashboard/
