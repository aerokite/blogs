
When **RBAC** is enabled in your Kubernetes cluster, you need **`Role`** which contains a set of permissions for your application deployed. Otherwise, your application will be failed to call any API resource (via API calls). 

You can provide your application full access permission. 

    rules:
    - apiGroups:
      - '*'
      resources:
      - '*'
      verbs:
      - '*'

But this is not the purpose of RBAC. Best if you can provide your application only needed permissions. Your application may need lots of permission. Finding those specific permisions can be difficult.

In this blog post, we will see how can we generate **`Role`** that will be need in your application for full functionality. 

We will use 

 - [**minikube**](https://kubernetes.io/docs/tasks/tools/install-minikube/) version: v0.24.1
 - [**audit2rbac**](https://github.com/liggitt/audit2rbac) version v0.5.0

Lets start minikube

    minikube start \
    --kubernetes-version=v1.9.0 \
    --bootstrapper=kubeadm --extra-config=apiserver.admission-control="NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ValidatingAdmissionWebhook,ResourceQuota,DefaultTolerationSeconds,MutatingAdmissionWebhook" \
    --mount --mount-string="$HOME/.minikube/files:/.minikube/files" \
    --feature-gates=AdvancedAuditing=true

**RBAC** is enabled in this minikube. And also local directory `~/.minikube/files`  is mounted with minikube directory `/.minikube/files`. So that we can easily use this audit log later.

Write `audit-policy.yaml` in `~/.minikube/files/audit` 

    apiVersion: audit.k8s.io/v1beta1
    kind: Policy
    rules:
    - level: Metadata

This **`Policy`** will be used in apiserver.

`ssh` into minikube to edit `/etc/kubernetes/manifests/kube-apiserver.yaml`

    $ minikube ssh
    $ cd /etc/kubernetes/manifests
    $ sudo vi kube-apiserver.yaml

We have to pass following flags to apiserver to for advanced audit. 

    # A file with the policy. See more (https://kubernetes.io/docs/tasks/debug-application-cluster/audit/#audit-policy)
    --audit-policy-file=/audit/audit-policy.yaml
    # Specifies the log file path that log backend uses to write audit events
    --audit-log-path=/audit/log/audit.log

And mount volume for `audit-policy.yaml` & `audit.log`

See what you need to edit [here](https://www.diffchecker.com/PeaXnSVm)

In container, directory `/audit` is mounter with minikube host directory `/.minikube/files/audit`.  This host directory `/.minikube/files` is already mounted with local directory `~/.minikube/files`. 

After done editing, apiserver will be restarted. We will see `audit.log` in `~/.minikube/files/audit/log` directory.

So our kubernetes cluster is generating audit events and we will use this log file to generate **`Role`**.

In this blog post, we will see how can we generate **`Role`** for [kubedb operator](https://github.com/kubedb/operator). 

Lets create `ClusterRole` , `ClusterRoleBinding` and `ServiceAccount` with all access permission.

Create a cluster role with this YAML.

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: kubedb-operator
    rules:
    - apiGroups:
      - '*'
      resources:
      - '*'
      verbs:
      - '*'

Create a service account to use in your application.

    $ kubectl create serviceaccount -n kube-system kubedb-operator

Create a cluster role binding

    $ kubectl create clusterrolebinding kubedb-operator --clusterrole=kubedb-operator --serviceaccount=kube-system:kubedb-operator

Now create a deployment for **kubedb operator** 

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: kubedb-operator
      namespace: kube-system
    spec:
      replicas: 1
      selector:
        matchLabels:
          run: kubedb-operator
      template:
        metadata:
          labels:
            run: kubedb-operator
        spec:
          containers:
          - args:
            - run
            - --rbac=true
            image: kubedb/operator:0.8.0-v1beta1
            name: operator
            ports:
            - containerPort: 8080
              name: web
              protocol: TCP
          serviceAccountName: kubedb-operator

Wait for running pod.

    $ kubectl get pods -n kube-system --selector='run=kubedb-operator' --watch
    NAME                               READY     STATUS              RESTARTS   AGE
    kubedb-operator-5956bc7dd4-c7fnh   1/1       ContainerCreating   0          58s
    kubedb-operator-5956bc7dd4-c7fnh   1/1       Running             0          32s
    ^C⏎

When this application is running, audit backend will generate some log events. We will use those events to create **`Role`**

    $ audit2rbac -f audit.log --serviceaccount kube-system:kubedb-operator > kubedb-rbac.yaml

This `kubedb-rbac.yaml` will contain `ClusterRole`  and `ClusterRoleBinding` 

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: audit2rbac:system:serviceaccount:kube-system:kubedb-operator
    rules:
    - apiGroups:
      - ""
      resources:
      - nodes
      verbs:
      - get
      - list
      - watch
    - apiGroups:
      - apiextensions.k8s.io
      resources:
      - customresourcedefinitions
      verbs:
      - create
      - get
    - apiGroups:
      - kubedb.com
      resources:
      - dormantdatabases
      - elasticsearchs
      - memcacheds
      - mongodbs
      - mysqls
      - postgreses
      - redises
      - snapshots
      verbs:
      - get
      - list
      - watch
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: audit2rbac:system:serviceaccount:kube-system:kubedb-operator
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: audit2rbac:system:serviceaccount:kube-system:kubedb-operator
    subjects:
    - kind: ServiceAccount
      name: kubedb-operator
      namespace: kube-system
