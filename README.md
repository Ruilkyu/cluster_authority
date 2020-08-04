# cluster_authority
kubernetes集群用户权限梳理

## 用户分类
```
1、由Kubernetes管理的service account
2、是普通用户（普通用户被假定为由外部独立服务管理）
```


## master接口访问步骤
```
1、认证
authenation
2、授权
authorization
3、准入控制
admission control
```

## 一、认证
### service accounts
```
1、介绍：
Service Account是用来访问kubernetes API，通过kubernetes API创建和管理，每个account只能在一个namespace上生效，存储在kubernetes API中的Secrets资源。kubernetes 会默认创建，并且会自动挂载到Pod中的/run/secrets/kubernetes.io/serviceaccount的目录下
Service Account作为凭证而存储在Secret，这些凭证同时被挂载到pod中，从而允许pod与kubernetes API之间的调用

2、查看所有service accounts
kubectl get sa  --all-namespaces

NAMESPACE         NAME                 SECRETS   AGE
default           default              1         42d
kube-node-lease   default              1         42d
kube-public       default              1         42d
kube-system       admin-user           1         14d
kube-system       coredns              1         6d2h
kube-system       default              1         42d
kube-system       kube-state-metrics   1         4d23h
lotus             default              1         42d
monitoring        default              1         7d2h
```

## 二、授权
RBAC（Role-Based Access Control）
```
所有的权限都围绕角色展开，角色本身会包含一系列都权限规则，表明某个角色能做哪些事情。
比如管理员可以操作所有的资源，某个namespace的用户只能修改该namespace的内容，或者有些角色只允许读取资源。
```

### 1.角色Role
```
一个Role对象只能用于授予对某个单一命名空间中资源的访问权限
1.1 创建role
例如：
在default命名空间内创建一个具有pod读权限的Role对象

kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default
  name: pod-reader-only
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

1.2 查看集群所有Role
kubectl get role --all-namespaces

NAMESPACE     NAME                                             AGE
default       pod-reader-only                                  4s
kube-public   system:controller:bootstrap-signer               42d
kube-system   extension-apiserver-authentication-reader        42d
kube-system   system::leader-locking-kube-controller-manager   42d
kube-system   system::leader-locking-kube-scheduler            42d
kube-system   system:controller:bootstrap-signer               42d
kube-system   system:controller:cloud-provider                 42d
kube-system   system:controller:token-cleaner                  42d
```

### 2、集群角色ClusterRole
```
ClusterRole对象可以授予与Role对象相同的权限，但由于它们属于集群范围对象， 也可以使用它们授予对以下几种资源的访问权限：
1、集群范围资源（例如节点，即node）
2、非资源类型endpoint（例如”/healthz”）
3、跨所有命名空间的命名空间范围资源（例如pod，需要运行命令kubectl get pods --all-namespaces来查询集群中所有的pod）

2.1 创建ClusterRole
例如：
ClusterRole定义可用于授予用户对所有命名空间中的secret读访问权限

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: secret-reader-only
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]

2.2 查看集群所有ClusterRole
kubectl get clusterrole --all-namespaces

NAME                                                                   AGE
admin                                                                  42d
cluster-admin                                                          42d
edit                                                                   42d
kube-state-metrics                                                     5d
prometheus                                                             7d3h
secret-reader-only                                                     5s
system:aggregate-to-admin                                              42d
system:aggregate-to-edit                                               42d
system:aggregate-to-view                                               42d
system:auth-delegator                                                  42d
system:basic-user                                                      42d
system:certificates.k8s.io:certificatesigningrequests:nodeclient       42d
system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   42d
system:controller:attachdetach-controller                              42d
system:controller:certificate-controller                               42d
system:controller:clusterrole-aggregation-controller                   42d
system:controller:cronjob-controller                                   42d
system:controller:daemon-set-controller                                42d
system:controller:deployment-controller                                42d
system:controller:disruption-controller                                42d
system:controller:endpoint-controller                                  42d
system:controller:expand-controller                                    42d
system:controller:generic-garbage-collector                            42d
system:controller:horizontal-pod-autoscaler                            42d
system:controller:job-controller                                       42d
system:controller:namespace-controller                                 42d
system:controller:node-controller                                      42d
system:controller:persistent-volume-binder                             42d
system:controller:pod-garbage-collector                                42d
system:controller:pv-protection-controller                             42d
system:controller:pvc-protection-controller                            42d
system:controller:replicaset-controller                                42d
system:controller:replication-controller                               42d
system:controller:resourcequota-controller                             42d
system:controller:route-controller                                     42d
system:controller:service-account-controller                           42d
system:controller:service-controller                                   42d
system:controller:statefulset-controller                               42d
system:controller:ttl-controller                                       42d
system:coredns                                                         6d3h
system:csi-external-attacher                                           42d
system:csi-external-provisioner                                        42d
system:discovery                                                       42d
system:heapster                                                        42d
system:kube-aggregator                                                 42d
system:kube-controller-manager                                         42d
system:kube-dns                                                        42d
system:kube-scheduler                                                  42d
system:kubelet-api-admin                                               42d
system:node                                                            42d
system:node-bootstrapper                                               42d
system:node-problem-detector                                           42d
system:node-proxier                                                    42d
system:persistent-volume-provisioner                                   42d
system:public-info-viewer                                              42d
system:volume-scheduler                                                42d
view                                                                   42d
```

### 3、角色绑定
```
角色绑定包含了一组相关主体（即subject, 包括用户——User、用户组——Group、或者服务账户——Service Account）以及对被授予角色的引用

在命名空间中可以通过RoleBinding对象授予权限，而集群范围的权限授予则通过ClusterRoleBinding对象完成

3.1 普通角色绑定
例如：
RoleBinding可以引用在同一命名空间内定义的Role对象，将default命名空间中的pod-reader-only角色授予用户xiaoming,这一授权将允许用户xiaoming从default命名空间中读取pod
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: pod-reader-only
  namespace: default
subjects:
- kind: User
  name: xiaoming
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader-only
  apiGroup: rbac.authorization.k8s.io
3.2 系统角色绑定
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
   name: admin-user
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: cluster-admin

3.3 查看集群所有RoleBinding
kubectl get rolebinding --all-namespaces

NAMESPACE     NAME                                                AGE
default       pod-reader-only                                     91s
kube-public   system:controller:bootstrap-signer                  42d
kube-system   system::extension-apiserver-authentication-reader   42d
kube-system   system::leader-locking-kube-controller-manager      42d
kube-system   system::leader-locking-kube-scheduler               42d
kube-system   system:controller:bootstrap-signer                  42d
kube-system   system:controller:cloud-provider                    42d
kube-system   system:controller:token-cleaner                     42d

3.4 查看集群所有ClusterRoleBinding 
kubectl get clusterrolebinding --all-namespaces

NAME                                                   AGE
admin-user                                             14d
cluster-admin                                          42d
cluster-system-anonymous                               42d
kube-state-metrics                                     5d
kubelet-bootstrap                                      42d
prometheus                                             7d3h
system:anonymous                                       7d2h
system:basic-user                                      42d
system:controller:attachdetach-controller              42d
system:controller:certificate-controller               42d
system:controller:clusterrole-aggregation-controller   42d
system:controller:cronjob-controller                   42d
system:controller:daemon-set-controller                42d
system:controller:deployment-controller                42d
system:controller:disruption-controller                42d
system:controller:endpoint-controller                  42d
system:controller:expand-controller                    42d
system:controller:generic-garbage-collector            42d
system:controller:horizontal-pod-autoscaler            42d
system:controller:job-controller                       42d
system:controller:namespace-controller                 42d
system:controller:node-controller                      42d
system:controller:persistent-volume-binder             42d
system:controller:pod-garbage-collector                42d
system:controller:pv-protection-controller             42d
system:controller:pvc-protection-controller            42d
system:controller:replicaset-controller                42d
system:controller:replication-controller               42d
system:controller:resourcequota-controller             42d
system:controller:route-controller                     42d
system:controller:service-account-controller           42d
system:controller:service-controller                   42d
system:controller:statefulset-controller               42d
system:controller:ttl-controller                       42d
system:coredns                                         6d3h
system:discovery                                       42d
system:kube-controller-manager                         42d
system:kube-dns                                        42d
system:kube-scheduler                                  42d
system:node                                            42d
system:node-proxier                                    42d
system:public-info-viewer                              42d
system:volume-scheduler                                42d
```
