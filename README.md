# cks
cks exam related
根据网友给出的资料https://blog.csdn.net/pooopun/article/details/138262165
CKS认证考试共有16道题目，我们针对这16道题目，结合K8S文档，进行准备

# Q1 kube bench修复不安全项
context： 针对kubeadm创建的cluster运行CIS基准测试工具时，发现多个必须立刻解决的问题。
本题涉及到kube bench。
## 修改1：修改apiserver的配置文件。/etc/kubernetes/manifests/kube-apiserver.yaml文件
--authorization-mode #修改为Node，RBAC
## 修改2：修改kubelet配置文件 /var/lib/kubelet/config.yaml文件
![图片](https://github.com/user-attachments/assets/145892ed-c456-40ac-8072-991bf0a0ee2c)
authentication.anonymous.enabled: false #修改为false后，允许匿名用户登录。
authorization.mode: webhook #修改为webhook
k8s在线文档对于kubelet认证和授权的介绍网页如下：
https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/

##修改3：修改/etc/kubernetes/manifests/etcd.yaml文件
![图片](https://github.com/user-attachments/assets/b25010b5-de82-402b-88e3-0f2a07ed7845)
- --client-cert-auth=true #修改为 true
systemctl daemon-reload
systemctl restart kubelet



# Q2 pod指定serviceaccount
问题：
您组织的安全策略包括：
1，serviceaccount不得挂载api凭据
2，serviceaccount以-sa结尾
清单文件/cks/sa/pod1.yaml中指定的pod由于serviceaccount指定错误而无法调度。
请完成以下项目：
1，在现有namespace qa中创建一个名为backend-sa的新serviceaccount。确保该sa不自动挂载api凭证。
2，使用/cks/sa/pod1.yaml中的清单文件来创建一个pod。
3，清理namespace qa中未使用的service account。


## 1、创建SA vim qa-sa.yaml
vi qa-sa.yaml
kubectl apply -f qa-sa.yaml
--------------
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: qa
automountServiceAccountToken: false  //在线文档搜索automountserviceaccounttoken关键字，找到config SA for pod章节，其中有这段代码。
## 2，创建使用该sa的pod。
vi /cks/sa/pod1.yaml
kubectl apply -f pod1.yaml
kubectl get pod -n qa
-------------------
//this is the pod define file
apiVersion: v1
kind: Pod
metadata:
  name: backend //修改为要求的名字
  namespace: qa #注意命名空间是否正确
spec:
  serviceAccountName: backend-sa # 没有则添加一行，有则修改这一行为刚才创建的 ServiceAccount（考试时，默认已有这一行，需要修改。）
  containers: 
  - image: nginx:1.9
    imagePullPolicy: IfNotPresent
    name: backend
## 3, 删除namespace qa中没有使用的sa
kubectl get sa -n qa
kubectl get pod -n qa -o yaml > grep -i serviceaccountname  #找到所有的sa
kubectl delete sa xxx -n qa

# q3 默认网络策略
context：一个默认拒绝的网络策略可以避免在其他namespace中意外公开pod
task：
1，在namespace testing中，为所有类型为ingress+egress的流量创建名字为denypolicy的新默认拒绝网络策略。
2，这个网络策略必须拒绝所有ingress+egress流量
3，将这个网络策略应用到namespace testing中的所有pod上。
解题步骤：
# 1）创建networkpolicy文件
vi /cks/net/p1.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: denypolicy # 修改name
  namespace: testing # 添加namespace
spec:
  podSelector: {} #选中所有pod
  policyTypes: # ingress和egress只声明，没有内容。意思是不允许任何进入和外出的流量
  - Ingress	
  - Egress		
## 2）kubectl apply -f p1.yaml
## 3）kubectl describe networkpolicy denypolicy -n testing

#  q4 rbac rolebinding
context: 绑定到pod的serviceaccount被授予了过度宽松的权限，完成以下项目以减少权限
task
1）1个名为web-pod的pod已经在namespace db中运行。
2）编辑绑定到pod的serviceaccount service-account-web的role，仅允许对service类型的资源执行get操作。
3）在namespace db中创建一个role-2的role，只能对namespace资源执行delete操作。
4）创建一个名字为role-2-binding的新binding，将创建的role绑定到pod的serviceaccount。
注意：不要删除现有的rolebinding
## 解题步骤：
准备工作
1）kubectl describe rolebinding -n db
2) kubectl get role -n db

## 1)编辑role-1权限。//activate immediately
kubectl edit role role-1 -n db
rules: 		 #模拟环境里要删除掉 null，然后添加以下内容。考试时，要根据实际情况修改。
- apiGroups: [""] 
  resources: ["services"]
  verbs: ["get"] 
##  2)检查role权限更改后的结果
kubectl describe role role-1 -n db
##  3)创建名字为role-2的role，并绑定到sa
kubectl create role role-2 -n db --verb=delete --resource=namespace
kubectl create rolebinding role-2-binding -n db --serviceaccount=db:service-account-web --role=role-2

# Q5 log audit 日志审计
问题：在cluster中启用日志审计，并确保
1）日志存储到/var/log/kubernetes/audit-logs.txt
2）日志能保留10天
3）最多保留2个旧的日志文件
##解答步骤：
## 1) ssh root@master
## 2)配置审计策略
cp /etc/kubernetes/logpolicy/sample-policy.yaml /tmp
vi sample-policy.yaml 插入以下内容
  - level: RequestRespone
    resources:
    - group: ""
      resources: ["persistentvolumes"] 
  - level: Resquest
    resources:
    - group: ""
      resources: ["configmaps"] s
    namespace: ["front-apps"]
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets","configmaps"]
  - level: Metadata
    omitStages:
      - "RequestReceived"


## 3）配置master节点的api-server.yaml文件
cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp
vi kube-apiserver.yaml
    - --authorization-mode=Node,RBAC #在这一行的后面加参数如果考试中已经存在了则不要重复添加。
    /etc/kubernetes/logpolicy/sample-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit-logs.txt
    - --audit-log-maxage=10 
    - --audit-log-maxbackup=2
## 4)等待apiserver自动重启

## 5）检查
kubeclt get pod -A
tail /var/log/kubernetes/audit-logs.txt


# Q6 创建secret
task： 在namespace istio-system中获取名字为db1-tester的现有secret内容。
将username字段存储在/cks/sec/user.txt文件中，将password字段存储在/cks/sec/pass.txt中。
注意：你需要自己创建以上两个文件。
在istio-system namespace中创建名字为db2-test的secret，内容为：
username:
password:

最后，创建一个新的pod，通过卷访问secret db2-test.
pod名称：db2-test
namespace: istio-system
container: dve-container
image: nginx
volume name: secret-volume
path: /etc/secret




Q14 podSecurityPolicy
1, create PSP
this feature has been removed.
-----------------------Q15 api server authentication
删除clusterrolebinding，取消匿名用户的集群管理员权限。
kubectl delete clusterrolebinding system:anonymous
---------Q16 image policy web hook
https://developer.aliyun.com/article/1398152
1, mkdir -p /etc/kubernetes/epconfig
vim /etc/kubernetes/epconfig/admission_configuration.json
//设置image的策略
{
  "imagePolicy": {
     "kubeConfigFile": "/etc/kubernetes/kube-image-bouncer.yml",
     "allowTTL": 50,
     "denyTTL": 50,
     "retryBackoff": 500,
     "defaultAllow": true
  }
}

2, vim /etc/kubernetes/epconfig/kubeconfig.yaml
//是指集群和上下文。
apiVersion: v1
kind: Config
# clusters refers to the remote service.
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/epconfig/external-cert.pem  # CA for verifying the remote service.
    server: server  # URL of remote service to query. Must use 'https'.
  name: image-checker
contexts:
- context:
    cluster: image-checker
    user: api-server
  name: image-checker
current-context: image-checker
preferences: {}
# users refers to the API server's webhook configuration.
users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/epconfig/apiserver-client-cert.pem     # cert for the webhook admission controller to use
    client-key:  /etc/kubernetes/epconfig/apiserver-client-key.pem             # key matching the cert
3，测试验证文件
mkdir -p /cks/img/
vim /cks/img/web1.yaml           //

apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-latest
spec:
  replicas: 1
  selector:
    app: nginx-latest
  template:
    metadata:
      name: nginx-latest
      labels:
        app: nginx-latest
    spec:
      containers:
      - name: nginx-latest
        image: nginx
        ports:
        - containerPort: 80
-----------修改准入配置文件。
1，vim /etc/kubernetes/epconfig/adminssion_configuration.json
其中：defaultAllow: false
----------修改kube配置文件
1，vim /etc/kubernetes/epconfig/kubeconfig.yaml
在cluster字段下，添加server字段：
server: https://image-bouncer-webhook.default.svc:1323/image_policy
----------修改api-server配置文件
vim /etc/kubernetes/manifests/kube-apiserver.yaml
//启用imagePolicyWebHook插件，确认--admission-control-config-file已经配置并mount
--enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
--admission-control-config-file=/etc/kubernetes/epconfig/adminssion_configuration.json
重启kubelet
systemctl daemon-reload
systemctl restart kubelet
检查
1，kubectl create -f /cks/img/web1.yaml
kubectl describe rc nginx-latest
在输出结果的最后event字段中，可以看到is forbidden。
