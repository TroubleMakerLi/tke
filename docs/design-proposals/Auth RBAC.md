# Auth RBAC


**Author**:
jason ([@wangao1236](https://github.com/wangao1236))
mingyu  ([@metang326](https://github.com/metang326))


## Abstract

目前tke不支持k8s原生的rbac鉴权模式，无法通过创建ClusterRoleBinding、ClusterRole设置clusters、clustercredentials等资源的访问权限。


## Motivation

- 支持k8s原生的rbac鉴权

## Main proposal

增加一种鉴权方式，如果rbac鉴权通过则允许访问相应资源。

## Solution
1. 创建ClusterRoleBindings，将tke命名空间下默认创建的ServiceAccount（name：default）与ClusterRole（name：cluster-admin）进行绑定，使其能获得访问所有资源的权限。

2. 增加k8sinformers，对Roles、RoleBindings、ClusterRoles、ClusterRoleBindings进行watch。

3. 增加rbacAuthorizer，根据rbac进行鉴权，如对请求资源有相应权限则Allowed为true。

4. 对system用户跳过租户信息检查（CheckClustersTenant）。

原因：创建ServiceAccount时没有指定tenantID，所以传入auth鉴权时tenantID字段是空的，在没有指定具体对象、执行 kubectl get cls这样的操作时是可以正常执行的；但如果指定单个对象进行操作，比如kubectl get cls --kubeconfig=test.config cls-r5ppcm4s， cls-r5ppcm4s这个对象是有tenantID的，其tenantID字段为default。所以两者就表现为不匹配，未进行鉴权就返回了。报错信息如下：
```
[root@VM-187-235-centos ~/.kube/test]# kubectl get cls --kubeconfig=test.config
NAME           CREATED AT
cls-r5ppcm4s   2021-05-06T02:51:24Z
global         2021-02-23T06:49:58Z
[root@VM-187-235-centos ~/.kube/test]# kubectl get cls --kubeconfig=test.config cls-r5ppcm4s
Error from server (Forbidden): cluster:cls-r5ppcm4s.platform.tkestack.io "cls-r5ppcm4s" is forbidden: User "system:serviceaccount:tmy:rbac" cannot getCluster resource "cluster:cls-r5ppcm4s" in API group "platform.tkestack.io" at the cluster scope: cluster: cls-r5ppcm4s has invalid tenantID
[root@VM-187-235-centos ~/.kube/test]# kubectl edit cls --kubeconfig=test.config cls-r5ppcm4s
Error from server (Forbidden): cluster:cls-r5ppcm4s.platform.tkestack.io "cls-r5ppcm4s" is forbidden: User "system:serviceaccount:tmy:rbac" cannot getCluster resource "cluster:cls-r5ppcm4s" in API group "platform.tkestack.io" at the cluster scope: cluster: cls-r5ppcm4s has invalid tenantID
[root@VM-187-235-centos ~/.kube/test]#
[root@VM-187-235-centos ~/.kube/test]# kubectl edit cls --kubeconfig=test.config cls-r5ppcm4s -v=6
I0506 14:48:01.395073   24685 loader.go:379] Config loaded from file:  test.config
I0506 14:48:01.409034   24685 round_trippers.go:445] GET https://127.0.0.1:6443/apis/platform.tkestack.io/v1/clusters/cls-r5ppcm4s 403 Forbidden in 8 milliseconds
I0506 14:48:01.409290   24685 helpers.go:216] server response object: [{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "cluster:cls-r5ppcm4s.platform.tkestack.io \"cls-r5ppcm4s\" is forbidden: User \"system:serviceaccount:tmy:rbac\" cannot getCluster resource \"cluster:cls-r5ppcm4s\" in API group \"platform.tkestack.io\" at the cluster scope: cluster: cls-r5ppcm4s has invalid tenantID",
  "reason": "Forbidden",
  "details": {
    "name": "cls-r5ppcm4s",
    "group": "platform.tkestack.io",
    "kind": "cluster:cls-r5ppcm4s"
  },
  "code": 403
}]
```

5. 在tke-auth-api的configmap abac-policy.json里增加配置，允许system用户只读访问nonResourcePath。

原因：以ServiceAccount身份执行kubectl --kubeconfig=default-config get clusters命令时，首先需要知道clusters这个对象是哪个apiGroups里面的，然后才会去访问clusters这个资源。所以需要先list path "/apis/platform.tkestack.io/v1，但绑定clusterrole时，只给serviceaccount赋予了 get clusters权限，没赋予list path的权限，因此这里报错the server doesn't have a resource type "clusters"，还没进入rbac鉴权就报错返回了。报错信息如下：
```
# kubectl --kubeconfig=default-config  get clusters  -v=5
I0506 12:48:14.025639    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/monitor.tkestack.io/v1": permission for list on /apis/monitor.tkestack.io/v1 not verify
I0506 12:48:14.032706    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/notify.tkestack.io/v1": permission for list on /apis/notify.tkestack.io/v1 not verify
I0506 12:48:14.039766    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/registry.tkestack.io/v1": permission for list on /apis/registry.tkestack.io/v1 not verify
I0506 12:48:14.045961    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/platform.tkestack.io/v1": permission for list on /apis/platform.tkestack.io/v1 not verify
I0506 12:48:14.074744    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/business.tkestack.io/v1": permission for list on /apis/business.tkestack.io/v1 not verify
I0506 12:48:14.087278    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/platform.tkestack.io/v1": permission for list on /apis/platform.tkestack.io/v1 not verify
I0506 12:48:14.090143    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/registry.tkestack.io/v1": permission for list on /apis/registry.tkestack.io/v1 not verify
I0506 12:48:14.102832    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/monitor.tkestack.io/v1": permission for list on /apis/monitor.tkestack.io/v1 not verify
I0506 12:48:14.105774    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/notify.tkestack.io/v1": permission for list on /apis/notify.tkestack.io/v1 not verify
I0506 12:48:14.108652    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/business.tkestack.io/v1": permission for list on /apis/business.tkestack.io/v1 not verify
I0506 12:48:14.108687    5238 shortcut.go:89] Error loading discovery information: unable to retrieve the complete list of server APIs: business.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/business.tkestack.io/v1": permission for list on /apis/business.tkestack.io/v1 not verify, monitor.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/monitor.tkestack.io/v1": permission for list on /apis/monitor.tkestack.io/v1 not verify, notify.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/notify.tkestack.io/v1": permission for list on /apis/notify.tkestack.io/v1 not verify, platform.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/platform.tkestack.io/v1": permission for list on /apis/platform.tkestack.io/v1 not verify, registry.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/registry.tkestack.io/v1": permission for list on /apis/registry.tkestack.io/v1 not verify
I0506 12:48:14.124916    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/monitor.tkestack.io/v1": permission for list on /apis/monitor.tkestack.io/v1 not verify
I0506 12:48:14.134435    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/registry.tkestack.io/v1": permission for list on /apis/registry.tkestack.io/v1 not verify
I0506 12:48:14.161989    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/notify.tkestack.io/v1": permission for list on /apis/notify.tkestack.io/v1 not verify
I0506 12:48:14.166121    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/platform.tkestack.io/v1": permission for list on /apis/platform.tkestack.io/v1 not verify
I0506 12:48:14.170005    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/business.tkestack.io/v1": permission for list on /apis/business.tkestack.io/v1 not verify
I0506 12:48:14.170633    5238 discovery.go:214] Invalidating discovery information
I0506 12:48:14.357987    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/registry.tkestack.io/v1": permission for list on /apis/registry.tkestack.io/v1 not verify
I0506 12:48:14.505355    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/business.tkestack.io/v1": permission for list on /apis/business.tkestack.io/v1 not verify
I0506 12:48:14.508706    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/monitor.tkestack.io/v1": permission for list on /apis/monitor.tkestack.io/v1 not verify
I0506 12:48:14.511965    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/notify.tkestack.io/v1": permission for list on /apis/notify.tkestack.io/v1 not verify
I0506 12:48:14.515279    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/platform.tkestack.io/v1": permission for list on /apis/platform.tkestack.io/v1 not verify
I0506 12:48:14.535858    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/notify.tkestack.io/v1": permission for list on /apis/notify.tkestack.io/v1 not verify
I0506 12:48:14.539121    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/registry.tkestack.io/v1": permission for list on /apis/registry.tkestack.io/v1 not verify
I0506 12:48:14.542003    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/monitor.tkestack.io/v1": permission for list on /apis/monitor.tkestack.io/v1 not verify
I0506 12:48:14.544764    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-1s9FcIF0ln7nrzDH4PDQkwBF9dOtenant-default" cannot list path "/apis/platform.tkestack.io/v1": permission for list on /apis/platform.tkestack.io/v1 not verify
I0506 12:48:14.550956    5238 cached_discovery.go:78] skipped caching discovery info due to forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/business.tkestack.io/v1": permission for list on /apis/business.tkestack.io/v1 not verify
I0506 12:48:14.550984    5238 shortcut.go:89] Error loading discovery information: unable to retrieve the complete list of server APIs: business.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/business.tkestack.io/v1": permission for list on /apis/business.tkestack.io/v1 not verify, monitor.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/monitor.tkestack.io/v1": permission for list on /apis/monitor.tkestack.io/v1 not verify, notify.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/notify.tkestack.io/v1": permission for list on /apis/notify.tkestack.io/v1 not verify, platform.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/platform.tkestack.io/v1": permission for list on /apis/platform.tkestack.io/v1 not verify, registry.tkestack.io/v1: forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list path "/apis/registry.tkestack.io/v1": permission for list on /apis/registry.tkestack.io/v1 not verify
error: the server doesn't have a resource type "clusters"
```
## Test
### step 0
在tmy这个命名空间下创建一个名为rbac-tenant-default的sa用于测试，其基本信息如下：
```
# kubectl get sa -ntmy rbac -oyaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbac-tenant-default
  namespace: tmy
secrets:
- name: rbac-token-5ljnn
```
获取其对应的secret后生成一个kubeconfig文件，用于后续的测试。
```
kubectl get secret -ntmy rbac-token-5ljnn -oyaml
echo -n "xxx" | base64 -d
cp ~/.kube/config test.config
vim test.config
```
test.config内容如下，其中token是上面执行echo -n "xxx" | base64 -d的结果。
```
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://127.0.0.1:6443
  name: default-cluster
contexts:
- context:
    cluster: default-cluster
    user: default-user
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users:
- name: default-user
  user:
    token: xxxx
```
创建clusterrole和clusterrolebinding，允许rbac对clusters进行get、list。
```
# kubectl get clusterrole cls-role -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cls-role
rules:
- apiGroups:
  - '*'
  resources:
  - clusters
  verbs:
  - get
  - list
# kubectl get clusterrolebinding cls-bind -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cls-bind
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cls-role
subjects:
- kind: ServiceAccount
  name: rbac-tenant-default
  namespace: tmy
```

# step 1
使用rbac的kubeconfig文件执行kubectl命令，clusters资源被授权因此可以访问；clustercredentials资源未被授权，因此不能访问。
```
# kubectl get clusters --kubeconfig=test.config
NAME           CREATED AT
cls-mmxx4mkr   2021-04-22T03:29:24Z
global         2021-02-23T06:49:58Z
# kubectl get clustercredentials  --kubeconfig=test.config
Error from server (Forbidden): clustercredentials.platform.tkestack.io is forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list resource "clustercredentials" in API group "platform.tkestack.io" at the cluster scope: permission for list on clustercredentials not verify
```

# step 2
通过kubectl edit clusterrole cls-role增加对clustercredentials的访问权限，修改后如下：
```
# kubectl get clusterrole cls-role -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cls-role
rules:
- apiGroups:
  - '*'
  resources:
  - clusters
  - clustercredentials
  verbs:
  - get
  - list
```
增加权限后，可访问clustercredentials资源。
```
# kubectl get clustercredentials  --kubeconfig=test.config
NAME          CREATED AT
cc-global     2021-02-23T06:49:58Z
cc-xsvfnrng   2021-04-22T03:29:24Z
```

# step 3
测试对pod、deployment等原生资源的访问。
未添加时:
```
# kubectl get pods  --kubeconfig=default-config
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list resource "pods" in API group "" in the namespace "default": permission for list on pods not verify
```
执行kubectl edit clusterrole cls-role，添加相关操作。
```
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - patch
  - get
  - list
  - create
  - delete
  - edit
```
添加后：
```
# kubectl edit clusterrole cls-role
clusterrole.rbac.authorization.k8s.io/cls-role edited
[root@VM-187-235-centos ~/428-test-tenant]# kubectl get pods  --kubeconfig=default-config
NAME                           READY   STATUS    RESTARTS   AGE
mingyu-test-865c78c6b4-5qxzq   1/1     Running   0          56d
mingyu-test-865c78c6b4-96qp2   1/1     Running   0          56d
```

deployments也是类似的操作。
```
# kubectl get deployments  --kubeconfig=default-config
Error from server (Forbidden): deployments.apps is forbidden: User "system:serviceaccount:tmy:rbac-tenant-default" cannot list resource "deployments" in API group "apps" in the namespace "default": permission for list on deployments not verify
```
添加内容：
```
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - patch
  - get
  - list
  - create
  - delete
```
添加后：
```
# kubectl get deployments  --kubeconfig=default-config
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
mingyu-test   2/2     2            2           56d
```
