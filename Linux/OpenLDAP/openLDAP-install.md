## kubernetes中Nginx Ingress + Cert-Manager实现自动https


在Kubernetes集群中使用 HTTPS 协议，需要一个证书管理器、一个证书自动签发服务，主要通过 Ingress 来发布 HTTPS 服务，因此需要Ingress Controller并进行配置，启用 HTTPS 及其路由.

cert-manager作为一系列部署资源在Kubernetes集群中运行.

 CustomResourceDefinitions(CRD) 用于配置证书颁发机构和请求证书.

 

### 安装cert-manager

#### 使用常规清单安装

所有的资源(CRD cert-manager, namespace和webhook组件), 都包含在单个的YAML清单中.

安装CRD和cert-manager

```bash
# Kubernetes 1.16+
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml

# Kubernetes <1.16
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager-legacy.yaml
```
PS: 请根据k8s版本自行选择

#### 使用helm安装

1. 准备工作
```bash
#为cert-manager创建名称空间
$ kubectl create namespace cert-manager
#添加Jetstack Helm存储库
helm repo add jetstack https://charts.jetstack.io
#更新您的本地Helm图表存储库缓存
$ helm repo update
```
2. CustomResourceDefinition使用kubectl以下命令安装资源:
```bash
# Kubernetes 1.15+
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.crds.yaml

# Kubernetes <1.15
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager-legacy.crds.yaml
```

3. 也可以选择在进行helm安装时设置--set installCRDs=true
```bash
# Helm v3+
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.1.0 \
  # --set installCRDs=true
$ helm install \
  cert-manager --namespace cert-manager \
  --set ingressShim.defaultIssuerName=letsencrypt-prod \
  --set ingressShim.defaultIssuerKind=ClusterIssuer \
  --version v1.1.0 jetstack/cert-manager
  
# 上诉命令中的两个set;用于支持kubernetes.io/tls-acme: "true"annotation 来自动化 TLS
# Helm v2
$ helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v1.1.0 \
  jetstack/cert-manager \
  # --set installCRDs=true
```
4. 验证安装
```bash
$ kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-bcd4f8795-zpxsw               1/1     Running   0          2m
cert-manager-cainjector-78cbd59555-dhsp8   1/1     Running   0          2m
cert-manager-webhook-756d477cc4-j5mzt      1/1     Running   0          2m
```

5. 确认正确设置了证书管理器并能够颁发基本证书类型
```bash
$ cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

创建测试资源

```bash
$ kubectl apply -f test-resources.yaml
#检查新创建证书的状态。您可能需要等待几秒钟，然后cert-manager才能处理证书请求
$ kubectl describe certificate -n cert-manager-test
#最后清理测试资源
$ kubectl delete -f test-resources.yaml
```
### 创建证书签发服务

cert-manager安装之后, 需要定义cluster issuer 或者 issuer资源, 才能进行证书的颁发. 这些资源代表特定的签名机构. 
[说明请参考官方文档](https://cert-manager.io/docs/configuration/).
支持多种的签发类型,这里我们使用的ACME.

关于ACME有以下两种认证方式

-  HTTP01; 适用于有公网IP的ingress资源. 且配置简单
-  DNS01; 适用于内网的k8s集群,且没有公网IP,实现自动化,需借助第三方工具;当然cert-manager也是默认支持一些[域名服务商](https://cert-manager.io/docs/configuration/acme/dns01/)的,对于不支持的厂商也提供了通用的[webhook](https://cert-manager.io/docs/configuration/acme/dns01/)方法. 如果没有在列表中,可以自己通过example实现.


~~简单理解~~,HTTP01就是通过请求你需要申请证书的域名下的;例如: `www.example.com/.well-know/acme-challenge/CR-WBJ-XXXXXXXX` 的路径,去校验网站; 前提是此路径必须公网能正常访问,且域名解析已正确指向你的ingress的公网地址. 就是校验ingress的公网IP以及域名解析的指向都必须是正确的. 才能颁发证书.

~~简单理解~~,DNS01就是acme服务器会给你生成一个txt的解析,需要您正确添加到您的DNS管理后台当中,且当ACME服务器校验解析生效时,才能证明域名的归属权是属于您的,才能给您颁发证书.

#### HTTP01

配置ClusterIssuer

PS: letsencrypt-staging 这里官网提供的实例是临时, 如果采用此ClusterIssuer默认会生成暂存 Lets Encrypt 证书,暂存证书由 CN=Fake LE Intermediate X1 颁发,如果你得到此证书,也表示已自动申请成功.将`ClusterIssuer YAML` 中的 ACME 服务器替换为 `https://acme-v02.api.letsencrypt.org/directory`;并创建一个新的名字为letsencrypt-prod的ClusterIssuer;且邮箱也必须替换.

官方示例:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: user@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: example-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```
letsencrypt-prod:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: yang-li@live.cn
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-prod-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```
然后可直接创建这个Clusterissuer资源
```bash
$ kubectl create -f cluster-issuer.yaml
clusterissuer.certmanager.k8s.io "letsencrypt-prod" created
$ kubectl get clusterissuer
NAME               AGE
letsencrypt-prod   16s
```
这里由于我没有公网的ingress资源以及集群,这里我就不再进行测试了.

#### DNS01

这里由于我的k8s集群在内网,且ingress使用的是二层的metalb. 所以我只能使用DNS01这种ACME的校验方式进行证书的申请.

由于我们的域名解析服务使用的是阿里云的.这里我们通过官方链接找到了[alidns-webhook](https://github.com/pragkent/alidns-webhook) 按照官方的步骤,我们成功申请到了证书.

1. Install alidns-webhook
```bash
# Install alidns-webhook to cert-manager namespace. 
kubectl apply -f https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/bundle.yaml
```

2. Create secret contains alidns credentials

这里的要进行base64转码
```bash
apiVersion: v1
kind: Secret
metadata:
  name: alidns-secret
  namespace: cert-manager
data:
  access-key: YOUR_ACCESS_KEY
  secret-key: YOUR_SECRET_KEY
```
关于Accesskey 和 AccessSecret大家可以在自己的阿里云账户中创建,并进行DNS域名管理的相关授权,也可以给阿里云提工单.

3. Clusterissuer
```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Change to your letsencrypt email
    email: liyang@aixbx.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - dns01:
        webhook:
          groupName: acme.yourcompany.com
          solverName: alidns
          config:
            region: ""
            accessKeySecretRef:
              name: alidns-secret
              key: access-key
            secretKeySecretRef:
              name: alidns-secret
              key: secret-key
```

    PS: 如果替换groupName,则需要在开始的bundle.yaml文件中也进行替换.

4. 配置ingress

这里由于我的系统当中已经安装了kuboard的web-ui; 所以这里我们直接创建ingress即可.

[如何安装kuboard](https://github.com/easzlab/kubeasz/blob/master/docs/guide/kuboard.md)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuboard-dashboard
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: 'true' #强制HTTPS跳转
spec:
  tls:
  - hosts:
    - kuboard.it.aixbx.cn
    secretName: kuboard.it.aixbx.cn-tls
  rules:
  - host: kuboard.it.aixbx.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: kuboard
          servicePort: 80
```

以下排错和校验命令

```bash
$ kubectl get certificate
$ kubectl describe certificate NAME
$ kubectl describe secret NAME
$ kubectl describe order NAME
$ kubectl describe challenge NAME
```


最后的PS: 这里由于我内网的DNS服务器对我申请的域名证书进行了拦截,导致报错如下:

```
E1202 04:05:41.696710       1 sync.go:182] cert-manager/controller/challenges "msg"="propagation check failed" "error"="DNS record for \"kuboard.it.aixbx.cn\" not yet propagated" "dnsName"="kuboard.it.aixbx.cn" "resource_kind"="Challenge" "resource_name"="kuboard.it.aixbx.cn-tls-m5tmf-601210408-2135799246" "resource_namespace"="kube-system" "resource_version"="v1" "type"="DNS-01" 
```

 参考文档:
 - https://cert-manager.io/docs/installation/kubernetes/
 - https://cert-manager.io/docs/configuration/acme/
 - https://cert-manager.io/docs/configuration/acme/dns01/
 - https://github.com/pragkent/alidns-webhook
 - https://cert-manager.io/docs/tutorials/acme/ingress/
 - https://www.qikqiak.com/post/automatic-kubernetes-ingress-https-with-lets-encrypt/