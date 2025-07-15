

# 1. 踏み台サーバへSSHログイン

```
$ ssh lab-user@bastion.foo.bar.opentlc.com
```


# 2. install-config.yamlの作成

```
[lab-user@bastion ~]$ vi install-config.yaml
```

```
apiVersion: v1
baseDomain: sandboxXXX.opentlc.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform: 
    aws:
      type: m5.4xlarge # インスタンスタイプ
  replicas: 3 
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: demo
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: ap-northeast-1
publish: External
pullSecret: 'YOUR_PULL_SECRET'
```

# 3. OpenShiftをIPIでインストール

```
[lab-user@bastion ~]$ mkdir config
[lab-user@bastion ~]$ mv install-config.yaml config
[lab-user@bastion ~]$ openshift-install create cluster --dir=./config
? AWS Access Key ID xxx
? AWS Secret Access Key [? for help] ****************************************
...

INFO All cluster operators have completed progressing 
INFO Checking to see if there is a route at openshift-console/console... 
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 
INFO     export KUBECONFIG=/home/lab-user/config/auth/kubeconfig 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.demo.sandbox<XXX>.opentlc.com 
INFO Login to the console with user: "kubeadmin", and password: "<Passowrd>" 
INFO Time elapsed: 42m51s  
```

## インストール完了を確認

```
[lab-user@bastion ~]$ export KUBECONFIG=/home/lab-user/config/auth/kubeconfig
[lab-user@bastion ~]$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.18.19   True        False         False      10m     
baremetal                                  4.18.19   True        False         False      28m     
cloud-controller-manager                   4.18.19   True        False         False      31m     
cloud-credential                           4.18.19   True        False         False      33m     
cluster-autoscaler                         4.18.19   True        False         False      28m     
config-operator                            4.18.19   True        False         False      29m     
console                                    4.18.19   True        False         False      18m     
control-plane-machine-set                  4.18.19   True        False         False      25m     
csi-snapshot-controller                    4.18.19   True        False         False      29m     
dns                                        4.18.19   True        False         False      28m     
etcd                                       4.18.19   True        False         False      28m     
image-registry                             4.18.19   True        False         False      20m     
ingress                                    4.18.19   True        False         False      20m     
insights                                   4.18.19   True        False         False      28m     
kube-apiserver                             4.18.19   True        False         False      26m     
kube-controller-manager                    4.18.19   True        False         False      26m     
kube-scheduler                             4.18.19   True        False         False      25m     
kube-storage-version-migrator              4.18.19   True        False         False      29m     
machine-api                                4.18.19   True        False         False      25m     
machine-approver                           4.18.19   True        False         False      28m     
machine-config                             4.18.19   True        False         False      28m     
marketplace                                4.18.19   True        False         False      28m     
monitoring                                 4.18.19   True        False         False      17m     
network                                    4.18.19   True        False         False      30m     
node-tuning                                4.18.19   True        False         False      20m     
olm                                        4.18.19   True        False         False      19m     
openshift-apiserver                        4.18.19   True        False         False      21m     
openshift-controller-manager               4.18.19   True        False         False      24m     
openshift-samples                          4.18.19   True        False         False      20m     
operator-lifecycle-manager                 4.18.19   True        False         False      28m     
operator-lifecycle-manager-catalog         4.18.19   True        False         False      28m     
operator-lifecycle-manager-packageserver   4.18.19   True        False         False      21m     
service-ca                                 4.18.19   True        False         False      29m     
storage                                    4.18.19   True        False         False      28m

[lab-user@bastion ~]$ oc get machines -n openshift-machine-api
NAME                                      PHASE     TYPE         REGION           ZONE              AGE
demo-97sfb-master-0                       Running   m6i.xlarge   ap-northeast-1   ap-northeast-1a   34m
demo-97sfb-master-1                       Running   m6i.xlarge   ap-northeast-1   ap-northeast-1c   34m
demo-97sfb-master-2                       Running   m6i.xlarge   ap-northeast-1   ap-northeast-1d   34m
demo-97sfb-worker-ap-northeast-1a-tcrbk   Running   m5.4xlarge   ap-northeast-1   ap-northeast-1a   28m
demo-97sfb-worker-ap-northeast-1c-psjv9   Running   m5.4xlarge   ap-northeast-1   ap-northeast-1c   28m
demo-97sfb-worker-ap-northeast-1d-hgz2h   Running   m5.4xlarge   ap-northeast-1   ap-northeast-1d   28m
```

# AWSベアメタルインスタンスをクラスタに追加

```
[lab-user@bastion ~]$ export infrastructure_ID=$(oc get machineset -A | grep worker-ap-northeast-1a | awk '{print $2}' | sed 's/-worker.*//')
```

```
[lab-user@bastion ~]$ export ami_id=$(oc get configmap/coreos-bootimages -n openshift-machine-config-operator -o jsonpath='{.data.stream}' | jq -r '.architectures.x86_64.images.aws.regions."ap-northeast-1".image')
```

```
[lab-user@bastion ~]$ vi machine-bm.yaml
```

```
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: ${infrastructure_ID}-bm-worker-ap-northeast-1a
  namespace: openshift-machine-api
  labels:
    machine.openshift.io/cluster-api-cluster: ${infrastructure_ID}
spec:
  replicas: 2
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: ${infrastructure_ID}
      machine.openshift.io/cluster-api-machineset: ${infrastructure_ID}-bm-worker-ap-northeast-1a
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: ${infrastructure_ID}
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: ${infrastructure_ID}-bm-worker-ap-northeast-1a
    spec:
      lifecycleHooks: {}
      metadata:
        labels:
          node-role.kubernetes.io/worker: ""
      providerSpec:
        value:
          userDataSecret:
            name: worker-user-data
          placement:
            availabilityZone: ap-northeast-1a
            region: ap-northeast-1
          credentialsSecret:
            name: aws-cloud-credentials
          instanceType: c5n.metal
          metadata:
            creationTimestamp: null
          blockDevices:
            - ebs:
                encrypted: true
                iops: 0
                kmsKey:
                  arn: ''
                volumeSize: 120
                volumeType: gp3
          securityGroups:
            - filters:
                - name: 'tag:Name'
                  values:
                    - ${infrastructure_ID}-node
            - filters:
                - name: 'tag:Name'
                  values:
                    - ${infrastructure_ID}-lb
          kind: AWSMachineProviderConfig
          metadataServiceOptions: {}
          tags:
            - name: kubernetes.io/cluster/${infrastructure_ID}
              value: owned
          deviceIndex: 0
          ami:
            id: ${ami_id}
          subnet:
            filters:
              - name: 'tag:Name'
                values:
                  - ${infrastructure_ID}-subnet-private-ap-northeast-1a
          apiVersion: machine.openshift.io/v1beta1
          iamInstanceProfile:
            id: ${infrastructure_ID}-worker-profile
```

```
[lab-user@bastion ~]$ cat machine-bm.yaml | envsubst | oc apply -f -
[lab-user@bastion ~]$ oc get machineset -A
NAMESPACE               NAME                                   DESIRED   CURRENT   READY   AVAILABLE   AGE
openshift-machine-api   demo-97sfb-bm-worker-ap-northeast-1a   2         2         2       2           23m
openshift-machine-api   demo-97sfb-worker-ap-northeast-1a      1         1         1       1           62m
openshift-machine-api   demo-97sfb-worker-ap-northeast-1c      1         1         1       1           62m
openshift-machine-api   demo-97sfb-worker-ap-northeast-1d      1         1         1       1           62m
```

# OpenShift Data Foundationのインストール

# デフォルトストレージクラスの変更

# OpenShift Virtualizationのインストール