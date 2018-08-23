# 生产环境 k8s 使用指南

登录到具有 k8s 管理能力的节点：

- 已安装 `kubectl` 工具；
- 配置了能够直接 ssh 到 node 节点的 RSA 密钥；

```
➜  ~ k8sp
或者
➜  ~ ssh deployer@172.31.9.80
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-53-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

147 packages can be updated.
0 updates are security updates.


*** System restart required ***
Last login: Fri Jan 12 07:19:54 2018 from 172.31.10.66
[deployer@k8s-deploy-prod0]$
```

## kubectl 工具

> 版本略低，当前最新为 1.9 ；

```
[deployer@k8s-deploy-prod0]$ kubectl version
Client Version: version.Info{Major:"1", Minor:"4", GitVersion:"v1.4.6", GitCommit:"e569a27d02001e343cb68086bc06d47804f62af6", GitTreeState:"clean", BuildDate:"2016-11-12T05:22:15Z", GoVersion:"go1.6.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"4", GitVersion:"v1.4.6", GitCommit:"e569a27d02001e343cb68086bc06d47804f62af6", GitTreeState:"clean", BuildDate:"2016-11-12T05:16:27Z", GoVersion:"go1.6.3", Compiler:"gc", Platform:"linux/amd64"}
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$


# kubectl 使用
[deployer@k8s-deploy-prod0]$ kubectl -h
kubectl controls the Kubernetes cluster manager.

Find more information at https://github.com/kubernetes/kubernetes.

Basic Commands (Beginner):
  create         Create a resource by filename or stdin
  expose         Take a replication controller, service, deployment or pod and expose it as a new Kubernetes Service
  run            Run a particular image on the cluster
  set            Set specific features on objects

Basic Commands (Intermediate):
  get            Display one or many resources
  explain        Documentation of resources
  edit           Edit a resource on the server
  delete         Delete resources by filenames, stdin, resources and names, or by resources and label selector

Deploy Commands:
  rollout        Manage a deployment rollout
  rolling-update Perform a rolling update of the given ReplicationController
  scale          Set a new size for a Deployment, ReplicaSet, Replication Controller, or Job
  autoscale      Auto-scale a Deployment, ReplicaSet, or ReplicationController

Cluster Management Commands:
  cluster-info   Display cluster info
  top            Display Resource (CPU/Memory/Storage) usage
  cordon         Mark node as unschedulable
  uncordon       Mark node as schedulable
  drain          Drain node in preparation for maintenance
  taint          Update the taints on one or more nodes

Troubleshooting and Debugging Commands:
  describe       Show details of a specific resource or group of resources
  logs           Print the logs for a container in a pod
  attach         Attach to a running container
  exec           Execute a command in a container
  port-forward   Forward one or more local ports to a pod
  proxy          Run a proxy to the Kubernetes API server

Advanced Commands:
  apply          Apply a configuration to a resource by filename or stdin
  patch          Update field(s) of a resource using strategic merge patch
  replace        Replace a resource by filename or stdin
  convert        Convert config files between different API versions

Settings Commands:
  label          Update the labels on a resource
  annotate       Update the annotations on a resource
  completion     Output shell completion code for the given shell (bash or zsh)

Other Commands:
  api-versions   Print the supported API versions on the server, in the form of "group/version"
  config         Modify kubeconfig files
  help           Help about any command
  version        Print the client and server version information

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
[deployer@k8s-deploy-prod0]$


# kubectl get 使用
[deployer@k8s-deploy-prod0]$ kubectl get --help
Display one or many resources.

Valid resource types include:
   * clusters (valid only for federation apiservers)
   * componentstatuses (aka 'cs')
   * configmaps (aka 'cm')
   * daemonsets (aka 'ds')
   * deployments (aka 'deploy')
   * events (aka 'ev')
   * endpoints (aka 'ep')
   * horizontalpodautoscalers (aka 'hpa')
   * ingress (aka 'ing')
   * jobs
   * limitranges (aka 'limits')
   * nodes (aka 'no')
   * namespaces (aka 'ns')
   * petsets (alpha feature, may be unstable)
   * pods (aka 'po')
   * persistentvolumes (aka 'pv')
   * persistentvolumeclaims (aka 'pvc')
   * quota
   * resourcequotas (aka 'quota')
   * replicasets (aka 'rs')
   * replicationcontrollers (aka 'rc')
   * secrets
   * serviceaccounts (aka 'sa')
   * services (aka 'svc')


This command will hide resources that have completed. For instance, pods that are in the Succeeded or Failed phases.
You can see the full results for any resource by providing the '--show-all' flag.

By specifying the output as 'template' and providing a Go template as the value
of the --template flag, you can filter the attributes of the fetched resource(s).

Examples:
  # List all pods in ps output format.
  kubectl get pods

  # List all pods in ps output format with more information (such as node name).
  kubectl get pods -o wide

  # List a single replication controller with specified NAME in ps output format.
  kubectl get replicationcontroller web

  # List a single pod in JSON output format.
  kubectl get -o json pod web-pod-13je7

  # List a pod identified by type and name specified in "pod.yaml" in JSON output format.
  kubectl get -f pod.yaml -o json

  # Return only the phase value of the specified pod.
  kubectl get -o template pod/web-pod-13je7 --template={{.status.phase}}

  # List all replication controllers and services together in ps output format.
  kubectl get rc,services

  # List one or more resources by their type and names.
  kubectl get rc/web service/frontend pods/web-pod-13je7

Options:
      --all-namespaces=false: If present, list the requested object(s) across all namespaces. Namespace in current context is ignored even if specified with --namespace.
      --export=false: If true, use 'export' for the resources.  Exported resources are stripped of cluster-specific information.
  -f, --filename=[]: Filename, directory, or URL to a file identifying the resource to get from a server.
      --include-extended-apis=true: If true, include definitions of new APIs via calls to the API server. [default true]
  -L, --label-columns=[]: Accepts a comma separated list of labels that are going to be presented as columns. Names are case-sensitive. You can also use multiple flag options like -L label1 -L label2...
      --no-headers=false: When using the default or custom-column output format, don't print headers.
  -o, --output='': Output format. One of: json|yaml|wide|name|custom-columns=...|custom-columns-file=...|go-template=...|go-template-file=...|jsonpath=...|jsonpath-file=... See custom columns [http://kubernetes.io/docs/user-guide/kubectl-overview/#custom-columns], golang template [http://golang.org/pkg/text/template/#pkg-overview] and jsonpath template [http://kubernetes.io/docs/user-guide/jsonpath].
      --output-version='': Output the formatted object with the given group version (for ex: 'extensions/v1beta1').
      --raw='': Raw URI to request from the server.  Uses the transport specified by the kubeconfig file.
  -R, --recursive=false: Process the directory used in -f, --filename recursively. Useful when you want to manage related manifests organized within the same directory.
  -l, --selector='': Selector (label query) to filter on
  -a, --show-all=false: When printing, show all resources (default hide terminated pods.)
      --show-kind=false: If present, list the resource type for the requested object(s).
      --show-labels=false: When printing, show all labels as the last column (default hide labels column)
      --sort-by='': If non-empty, sort list types using this field specification.  The field specification is expressed as a JSONPath expression (e.g. '{.metadata.name}'). The field in the API resource specified by this JSONPath expression must be an integer or a string.
      --template='': Template string or path to template file to use when -o=go-template, -o=go-template-file. The template format is golang templates [http://golang.org/pkg/text/template/#pkg-overview].
  -w, --watch=false: After listing/getting the requested object, watch for changes.
      --watch-only=false: Watch for changes to the requested object(s), without listing/getting first.

Usage:
  kubectl get [(-o|--output=)json|yaml|wide|custom-columns=...|custom-columns-file=...|go-template=...|go-template-file=...|jsonpath=...|jsonpath-file=...] (TYPE [NAME | -l label] | TYPE/NAME ...) [flags] [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).
[deployer@k8s-deploy-prod0]$
```


## 获取 Nodes 信息

```
[deployer@k8s-deploy-prod0]$ alias k
alias k='kubectl --namespace=backend'
[deployer@k8s-deploy-prod0]$
```

### 获取全部 nodes 信息

```
[deployer@k8s-deploy-prod0]$ kubectl get nodes
NAME                                          STATUS    AGE
ip-172-1-32-165.cn-north-1.compute.internal   Ready     17d
ip-172-1-32-224.cn-north-1.compute.internal   Ready     16d
ip-172-1-33-129.cn-north-1.compute.internal   Ready     17d
ip-172-1-33-21.cn-north-1.compute.internal    Ready     18d
ip-172-1-33-79.cn-north-1.compute.internal    Ready     17d
ip-172-1-34-34.cn-north-1.compute.internal    Ready     16d
ip-172-1-35-165.cn-north-1.compute.internal   Ready     64d
ip-172-1-36-11.cn-north-1.compute.internal    Ready     16d
ip-172-1-36-132.cn-north-1.compute.internal   Ready     120d
ip-172-1-36-150.cn-north-1.compute.internal   Ready     16d
ip-172-1-37-106.cn-north-1.compute.internal   Ready     15d
ip-172-1-38-108.cn-north-1.compute.internal   Ready     217d
ip-172-1-38-210.cn-north-1.compute.internal   Ready     31d
ip-172-1-39-130.cn-north-1.compute.internal   Ready     50d
ip-172-1-39-237.cn-north-1.compute.internal   Ready     17d
ip-172-1-39-66.cn-north-1.compute.internal    Ready     120d
ip-172-1-40-133.cn-north-1.compute.internal   Ready     50d
ip-172-1-40-217.cn-north-1.compute.internal   Ready     18d
ip-172-1-40-92.cn-north-1.compute.internal    Ready     15d
ip-172-1-41-215.cn-north-1.compute.internal   Ready     64d
ip-172-1-42-14.cn-north-1.compute.internal    Ready     18d
ip-172-1-42-238.cn-north-1.compute.internal   Ready     15d
ip-172-1-43-146.cn-north-1.compute.internal   Ready     18d
ip-172-1-43-2.cn-north-1.compute.internal     Ready     120d
ip-172-1-43-33.cn-north-1.compute.internal    Ready     120d
ip-172-1-43-46.cn-north-1.compute.internal    Ready     120d
ip-172-1-43-56.cn-north-1.compute.internal    Ready     1d
ip-172-1-44-230.cn-north-1.compute.internal   Ready     15d
ip-172-1-44-48.cn-north-1.compute.internal    Ready     203d
ip-172-1-45-9.cn-north-1.compute.internal     Ready     18d
ip-172-1-46-244.cn-north-1.compute.internal   Ready     17d
ip-172-1-47-13.cn-north-1.compute.internal    Ready     43d
ip-172-1-47-41.cn-north-1.compute.internal    Ready     120d
ip-172-1-47-81.cn-north-1.compute.internal    Ready     64d
ip-172-1-48-164.cn-north-1.compute.internal   Ready     64d
ip-172-1-48-188.cn-north-1.compute.internal   Ready     17d
ip-172-1-48-191.cn-north-1.compute.internal   Ready     18d
ip-172-1-49-153.cn-north-1.compute.internal   Ready     15d
ip-172-1-50-116.cn-north-1.compute.internal   Ready     15d
ip-172-1-50-119.cn-north-1.compute.internal   Ready     217d
ip-172-1-50-19.cn-north-1.compute.internal    Ready     16d
ip-172-1-51-136.cn-north-1.compute.internal   Ready     15d
ip-172-1-52-100.cn-north-1.compute.internal   Ready     15d
ip-172-1-52-187.cn-north-1.compute.internal   Ready     15d
ip-172-1-53-147.cn-north-1.compute.internal   Ready     43d
ip-172-1-53-167.cn-north-1.compute.internal   Ready     18d
ip-172-1-55-121.cn-north-1.compute.internal   Ready     64d
ip-172-1-55-220.cn-north-1.compute.internal   Ready     15d
ip-172-1-55-247.cn-north-1.compute.internal   Ready     18d
ip-172-1-56-105.cn-north-1.compute.internal   Ready     16d
ip-172-1-56-178.cn-north-1.compute.internal   Ready     128d
ip-172-1-56-189.cn-north-1.compute.internal   Ready     6d
ip-172-1-56-22.cn-north-1.compute.internal    Ready     120d
ip-172-1-56-224.cn-north-1.compute.internal   Ready     120d
ip-172-1-56-8.cn-north-1.compute.internal     Ready     18d
ip-172-1-56-92.cn-north-1.compute.internal    Ready     15d
ip-172-1-57-20.cn-north-1.compute.internal    Ready     18d
ip-172-1-57-200.cn-north-1.compute.internal   Ready     120d
ip-172-1-57-69.cn-north-1.compute.internal    Ready     162d
ip-172-1-58-189.cn-north-1.compute.internal   Ready     16d
ip-172-1-58-80.cn-north-1.compute.internal    Ready     120d
ip-172-1-59-228.cn-north-1.compute.internal   Ready     16d
ip-172-1-59-67.cn-north-1.compute.internal    Ready     16d
ip-172-1-60-55.cn-north-1.compute.internal    Ready     17d
ip-172-1-61-102.cn-north-1.compute.internal   Ready     18d
ip-172-1-61-7.cn-north-1.compute.internal     Ready     1d
ip-172-1-62-114.cn-north-1.compute.internal   Ready     17d
ip-172-1-62-116.cn-north-1.compute.internal   Ready     16d
ip-172-1-63-49.cn-north-1.compute.internal    Ready     15d
ip-172-1-63-56.cn-north-1.compute.internal    Ready     50d
ip-172-1-63-70.cn-north-1.compute.internal    Ready     17d
ip-172-1-63-77.cn-north-1.compute.internal    Ready     17d
ip-172-1-63-9.cn-north-1.compute.internal     Ready     50d
[deployer@k8s-deploy-prod0]$
```

### 获取指定 node 信息

```
[deployer@k8s-deploy-prod0]$ kubectl get node ip-172-1-32-165.cn-north-1.compute.internal
NAME                                          STATUS    AGE
ip-172-1-32-165.cn-north-1.compute.internal   Ready     30d
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ kubectl get node ip-172-1-32-165.cn-north-1.compute.internal -o json
{
    "kind": "Node",
    "apiVersion": "v1",
    "metadata": {
        "name": "ip-172-1-32-165.cn-north-1.compute.internal",  -- 对应 nodeName 信息
        "selfLink": "/api/v1/nodes/ip-172-1-32-165.cn-north-1.compute.internal",
        "uid": "801a259e-e993-11e7-b481-02ab8baa063e",
        "resourceVersion": "394490119",
        "creationTimestamp": "2017-12-25T16:48:57Z",
        "labels": {
            "beta.kubernetes.io/arch": "amd64",
            "beta.kubernetes.io/instance-type": "c4.2xlarge",  -- EC2 机型
            "beta.kubernetes.io/os": "linux",
            "failure-domain.beta.kubernetes.io/region": "cn-north-1",
            "failure-domain.beta.kubernetes.io/zone": "cn-north-1a",
            "kubernetes.io/hostname": "ip-172-1-32-165.cn-north-1.compute.internal"
        },
        "annotations": {
            "volumes.kubernetes.io/controller-managed-attach-detach": "true"
        }
    },
    "spec": {
        "podCIDR": "100.97.70.0/24",    -- 限制跑在该 node 上的 pod 的 ip 模式
        "externalID": "i-0b562a9ec79013a92",
        "providerID": "aws:///cn-north-1a/i-0b562a9ec79013a92"
    },
    "status": {
        "capacity": {
            "alpha.kubernetes.io/nvidia-gpu": "0",
            "cpu": "8",
            "memory": "15403452Ki",
            "pods": "110"
        },
        "allocatable": {
            "alpha.kubernetes.io/nvidia-gpu": "0",
            "cpu": "8",
            "memory": "15403452Ki",
            "pods": "110"
        },
        "conditions": [
            {
                "type": "OutOfDisk",
                "status": "False",
                "lastHeartbeatTime": "2018-01-24T06:52:56Z",
                "lastTransitionTime": "2017-12-25T16:48:57Z",
                "reason": "KubeletHasSufficientDisk",
                "message": "kubelet has sufficient disk space available"
            },
            {
                "type": "MemoryPressure",
                "status": "False",
                "lastHeartbeatTime": "2018-01-24T06:52:56Z",
                "lastTransitionTime": "2017-12-25T16:48:57Z",
                "reason": "KubeletHasSufficientMemory",
                "message": "kubelet has sufficient memory available"
            },
            {
                "type": "DiskPressure",
                "status": "False",
                "lastHeartbeatTime": "2018-01-24T06:52:56Z",
                "lastTransitionTime": "2017-12-25T16:48:57Z",
                "reason": "KubeletHasNoDiskPressure",
                "message": "kubelet has no disk pressure"
            },
            {
                "type": "Ready",
                "status": "True",
                "lastHeartbeatTime": "2018-01-24T06:52:56Z",
                "lastTransitionTime": "2017-12-25T16:49:27Z",
                "reason": "KubeletReady",
                "message": "kubelet is posting ready status"
            },
            {
                "type": "NetworkUnavailable",
                "status": "False",
                "lastHeartbeatTime": "2018-01-24T06:52:52Z",
                "lastTransitionTime": "2018-01-24T06:52:52Z",
                "reason": "RouteCreated",
                "message": "RouteController created a route"
            }
        ],
        "addresses": [
            {
                "type": "InternalIP",
                "address": "172.1.32.165"   -- 当前 node 节点的内部访问 ip 地址
            },
            {
                "type": "LegacyHostIP",
                "address": "172.1.32.165"
            },
            {
                "type": "ExternalIP",
                "address": "54.222.210.35"  -- 当前 node 节点的外部访问 ip 地址
            }
        ],
        "daemonEndpoints": {
            "kubeletEndpoint": {
                "Port": 10250
            }
        },
        "nodeInfo": {
            "machineID": "e0b4079b66ec43cda2dd0a9af51dc3a7",
            "systemUUID": "EC277968-44F7-DF8D-F123-C8C7321BB3F0",
            "bootID": "104986ad-29db-4741-9bd4-1da9c3736b7b",
            "kernelVersion": "4.4.41-k8s",
            "osImage": "Debian GNU/Linux 8 (jessie)",
            "containerRuntimeVersion": "docker://1.11.2",
            "kubeletVersion": "v1.4.6",
            "kubeProxyVersion": "v1.4.6",
            "operatingSystem": "linux",
            "architecture": "amd64"
        },
        "images": [
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/fibre:0.1.1"
                ],
                "sizeBytes": 1540971783
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/study-performance:v0.0.74"
                ],
                "sizeBytes": 1480015162
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/cooper:v0.0.4"
                ],
                "sizeBytes": 1446383632
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.3.0_1.2.21"
                ],
                "sizeBytes": 1281848523
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.2.5_1.2.21"
                ],
                "sizeBytes": 1281840296
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.2.4_1.2.21"
                ],
                "sizeBytes": 1281840288
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.2.3_1.2.21"
                ],
                "sizeBytes": 1281825681
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.1.2_1.2.21"
                ],
                "sizeBytes": 1281707905
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.1.0_1.2.20"
                ],
                "sizeBytes": 1281707350
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.1.1_1.2.20"
                ],
                "sizeBytes": 1281707245
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.0.5_1.2.19"
                ],
                "sizeBytes": 1281678077
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.0.4_1.2.19"
                ],
                "sizeBytes": 1281676937
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.0.3_1.2.19"
                ],
                "sizeBytes": 1281673277
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.0.2_1.2.18"
                ],
                "sizeBytes": 1276128299
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.0.0_1.2.18"
                ],
                "sizeBytes": 1276126292
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:6.0.1_1.2.18"
                ],
                "sizeBytes": 1276126187
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:5.52.2_1.2.18"
                ],
                "sizeBytes": 1276076103
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:5.52.1_1.2.18"
                ],
                "sizeBytes": 1276075901
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/neo_production:5.52.0_1.2.18"
                ],
                "sizeBytes": 1276075901
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/course-center-service:17.0.1"
                ],
                "sizeBytes": 897852639
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/course-center-service:17.0.0"
                ],
                "sizeBytes": 897847127
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/course-center-service:13.0.1"
                ],
                "sizeBytes": 893970497
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/kensaku:master-df69be7e"
                ],
                "sizeBytes": 758325690
            },
            {
                "names": [
                    "prod-reg.llsops.com/platform-rls/grafana-syncr:master-21f458d7"
                ],
                "sizeBytes": 725241820
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/tape:v0.0.9"
                ],
                "sizeBytes": 722505628
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/russell:v0.1.0"
                ],
                "sizeBytes": 715877647
            },
            {
                "names": [
                    "\u003cnone\u003e@\u003cnone\u003e",
                    "\u003cnone\u003e:\u003cnone\u003e"
                ],
                "sizeBytes": 333040378
            },
            {
                "names": [
                    "prod-reg.llsops.com/library/k8s-fluentd-kafka-prod:1.19"
                ],
                "sizeBytes": 330714048
            },
            {
                "names": [
                    "gcr.io/google_containers/kube-proxy:v1.4.6"
                ],
                "sizeBytes": 202280808
            },
            {
                "names": [
                    "prod-reg.llsops.com/algorithm-rls/modelsrv:cc-0.3.21-prod"
                ],
                "sizeBytes": 162324597
            },
            {
                "names": [
                    "prod-reg.llsops.com/backend-rls/audcnv-service:v0.0.3"
                ],
                "sizeBytes": 138911499
            },
            {
                "names": [
                    "prod-reg.llsops.com/algorithm-rls/chatsrv:v3.15_test"
                ],
                "sizeBytes": 39466456
            },
            {
                "names": [
                    "gcr.io/google_containers/pause-amd64:3.0"
                ],
                "sizeBytes": 746888
            }
        ]
    }
}
```

> 上述内容主要包括：
> 
> - 当前 node 的名字和主机名（当前两者内容一致，且其中包含了 InternalIP 地址信息）
> - 当前 node 的 EC2 instance type
> - 当前 node 所在的 region 和 zone
> - 当前 node 上 pod 可用的地址模式 podCIDR
> - 当前 node 的一些系统基础指标信息
> - 当前 node 的内核版本、操作系统等信息
> - 当前 node 的 InternalIP/LegacyHostIP/ExternalIP 地址信息
> - 当期 node 上存在的 images 信息


通过 describe 查看指定 node 信息

```
[deployer@k8s-deploy-prod0]$ kubectl describe node ip-172-1-32-165.cn-north-1.compute.internal
Name:			ip-172-1-32-165.cn-north-1.compute.internal
Labels:			beta.kubernetes.io/arch=amd64
			beta.kubernetes.io/instance-type=c4.2xlarge
			beta.kubernetes.io/os=linux
			failure-domain.beta.kubernetes.io/region=cn-north-1
			failure-domain.beta.kubernetes.io/zone=cn-north-1a
			kubernetes.io/hostname=ip-172-1-32-165.cn-north-1.compute.internal
Taints:			<none>
CreationTimestamp:	Mon, 25 Dec 2017 16:48:57 +0000
Phase:
Conditions:
  Type			Status	LastHeartbeatTime			LastTransitionTime		Reason				Message
  ----			------	-----------------			------------------		------				-------
  OutOfDisk 		False 	Wed, 24 Jan 2018 08:04:41 +0000 	Mon, 25 Dec 2017 16:48:57 +0000 	KubeletHasSufficientDisk 	kubelet has sufficient disk space available
  MemoryPressure 	False 	Wed, 24 Jan 2018 08:04:41 +0000 	Mon, 25 Dec 2017 16:48:57 +0000 	KubeletHasSufficientMemory 	kubelet has sufficient memory available
  DiskPressure 		False 	Wed, 24 Jan 2018 08:04:41 +0000 	Mon, 25 Dec 2017 16:48:57 +0000 	KubeletHasNoDiskPressure 	kubelet has no disk pressure
  Ready 		True 	Wed, 24 Jan 2018 08:04:41 +0000 	Mon, 25 Dec 2017 16:49:27 +0000 	KubeletReady 			kubelet is posting ready status
  NetworkUnavailable 	False 	Wed, 24 Jan 2018 08:04:42 +0000 	Wed, 24 Jan 2018 08:04:42 +0000 	RouteCreated 			RouteController created a route
Addresses:		172.1.32.165,172.1.32.165,54.222.210.35
Capacity:
 alpha.kubernetes.io/nvidia-gpu:	0
 cpu:					8
 memory:				15403452Ki
 pods:					110
Allocatable:
 alpha.kubernetes.io/nvidia-gpu:	0
 cpu:					8
 memory:				15403452Ki
 pods:					110
System Info:
 Machine ID:			e0b4079b66ec43cda2dd0a9af51dc3a7
 System UUID:			EC277968-44F7-DF8D-F123-C8C7321BB3F0
 Boot ID:			104986ad-29db-4741-9bd4-1da9c3736b7b
 Kernel Version:		4.4.41-k8s
 OS Image:			Debian GNU/Linux 8 (jessie)
 Operating System:		linux
 Architecture:			amd64
 Container Runtime Version:	docker://1.11.2
 Kubelet Version:		v1.4.6
 Kube-Proxy Version:		v1.4.6
PodCIDR:			100.97.70.0/24
ExternalID:			i-0b562a9ec79013a92
Non-terminated Pods:		(10 in total)
  Namespace			Name								CPU Requests	CPU Limits	Memory Requests	Memory Limits
  ---------			----								------------	----------	---------------	-------------
  algorithm			barista-chatsrv-v002-lo2v0					0 (0%)		0 (0%)		0 (0%)		0 (0%)
  algorithm			lq-modelsrv-prod-v025-wadz3					0 (0%)		0 (0%)		0 (0%)		0 (0%)
  backend			fibre-prod-agent-v004-xpl06					200m (2%)	200m (2%)	300Mi (1%)	300Mi (1%)
  backend			kensaku-search-v008-keqib					1 (12%)		2 (25%)		2000Mi (13%)	2000Mi (13%)
  backend			neo-prod-api-v184-z5qom						1200m (15%)	1400m (17%)	2000Mi (13%)	2000Mi (13%)
  backend			neo-prod-extra-api-v181-04dhr					1 (12%)		1 (12%)		1500Mi (9%)	1500Mi (9%)
  backend			neo-prod-sidekiq-cc-v077-25k0c					1300m (16%)	1300m (16%)	2000Mi (13%)	2000Mi (13%)
  backend			tape-prod-kafka-v013-13ezg					500m (6%)	500m (6%)	400Mi (2%)	400Mi (2%)
  kube-system			fluentd-kafka-v1-605uj						100m (1%)	0 (0%)		200Mi (1%)	200Mi (1%)
  kube-system			kube-proxy-ip-172-1-32-165.cn-north-1.compute.internal		100m (1%)	0 (0%)		0 (0%)		0 (0%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.
  CPU Requests	CPU Limits	Memory Requests	Memory Limits
  ------------	----------	---------------	-------------
  5400m (67%)	6400m (80%)	8400Mi (55%)	8400Mi (55%)
No events.
[deployer@k8s-deploy-prod0]$
```

> 上述内容主要包括：
> 
> - 和 get 命令获得的信息几乎相同，但进行了格式化输出
> - 移除了当前 node 上存在的 images 信息
> - **能够显示当前 node 上运行的 Pods 信息**
> - 显示了 Requests/Limits 信息，以及 overcommitted 说明


### 登录到指定 node 上

```
[deployer@k8s-deploy-prod0]$ ll .ssh/
total 100
drwx------  2 deployer users  4096 Jan 17 02:55 ./
drwxr-xr-x 18 deployer users  4096 Jan 22 08:58 ../
-rw-------  1 deployer users 13068 Jan 17 02:55 authorized_keys
-rw-r--r--  1 deployer users   111 Sep 14 13:15 config
-rw-------  1 deployer users  1679 Dec 20  2016 id_rsa
-rw-------  1 deployer users  3247 Dec 20  2016 id_rsa_prod0-cluster-cn-north1a
-rw-r--r--  1 deployer users   745 Dec 20  2016 id_rsa_prod0-cluster-cn-north1a.pub -- 所有 k8s node 上均配置了该 public key 的内容
-rw-r--r--  1 deployer users   401 Dec 20  2016 id_rsa.pub   -- 内部包含 deployer@k8s-deploy 子串
-rw-------  1 deployer users 29522 Jan 24 03:00 known_hosts
-rw-r--r--  1 deployer users 17090 Aug  2 05:25 known_hosts.old
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ ssh admin@172.1.63.9 -i ~/.ssh/id_rsa_prod0-cluster-cn-north1a

# 也可以直接使用如下命令登录（利用 DNS 查询）
# ssh admin@ip-172-1-63-9.cn-north-1.compute.internal -i ~/.ssh/id_rsa_prod0-cluster-cn-north1a

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Jan 12 07:16:07 2018 from 172.31.9.80
admin@ip-172-1-63-9:~$
```


另外，在 `.ssh/config` 中找到 k8s master 节点的配置信息

```
[deployer@k8s-deploy-prod0]$ cat .ssh/config
Host k8s-prod0-master
	HostName 172.1.55.130
	User ubuntu
	IdentityFile ~/.ssh/id_rsa_prod0-cluster-cn-north1a
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ ssh k8s-prod0-master
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-53-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

125 packages can be updated.
0 updates are security updates.


*** System restart required ***
Last login: Wed Jan 24 03:20:44 2018 from 172.31.9.80
[ubuntu@ip-172-1-55-130]$
[ubuntu@ip-172-1-55-130]$ docker ps
Cannot connect to the Docker daemon. Is the docker daemon running on this host?
[ubuntu@ip-172-1-55-130]$
[ubuntu@ip-172-1-55-130]$ sudo docker ps
CONTAINER ID        IMAGE                                                     COMMAND                  CREATED             STATUS              PORTS               NAMES
4d698bb51ce7        gcr.io/google_containers/kube-scheduler:v1.4.6            "/bin/sh -c '/usr/loc"   13 months ago       Up 13 months                            k8s_kube-scheduler.8fd90df8_kube-scheduler-ip-172-1-55-130.cn-north-1.compute.internal_kube-system_c662244e94129e08cd89852a19722080_495c9c07
f3fd4fb750c5        gcr.io/google_containers/kube-controller-manager:v1.4.6   "/bin/sh -c '/usr/loc"   13 months ago       Up 13 months                            k8s_kube-controller-manager.697ed47d_kube-controller-manager-ip-172-1-55-130.cn-north-1.compute.internal_kube-system_ccd6c69acb0261a7bdc9e51f0aac887b_a02b9239
ec430baedf59        gcr.io/google_containers/kube-apiserver:v1.4.6            "/bin/sh -c '/usr/loc"   13 months ago       Up 13 months                            k8s_kube-apiserver.5184b6eb_kube-apiserver-ip-172-1-55-130.cn-north-1.compute.internal_kube-system_39bd723bcaae54657073e78e769f5666_68a12e60
09eb4cfe0279        gcr.io/google_containers/pause-amd64:3.0                  "/pause"                 13 months ago       Up 13 months                            k8s_POD.d8dbe16c_kube-scheduler-ip-172-1-55-130.cn-north-1.compute.internal_kube-system_c662244e94129e08cd89852a19722080_27c87cc6
b58ad546fa0c        gcr.io/google_containers/pause-amd64:3.0                  "/pause"                 13 months ago       Up 13 months                            k8s_POD.d8dbe16c_kube-controller-manager-ip-172-1-55-130.cn-north-1.compute.internal_kube-system_ccd6c69acb0261a7bdc9e51f0aac887b_beda6c9e
a853e050202f        gcr.io/google_containers/pause-amd64:3.0                  "/pause"                 13 months ago       Up 13 months                            k8s_POD.d8dbe16c_kube-apiserver-ip-172-1-55-130.cn-north-1.compute.internal_kube-system_39bd723bcaae54657073e78e769f5666_5444bd57
[ubuntu@ip-172-1-55-130]$
```

## 获取 Pods 信息

### 获取 default namespace 下的 pods 信息

```
[deployer@k8s-deploy-prod0]$ kubectl get pods
```

### 获取指定 namespace 下的 pods 信息

```
# 下面几种形式都能工作
[deployer@k8s-deploy-prod0]$ kubectl --namespace=backend get pods
[deployer@k8s-deploy-prod0]$ kubectl --namespace backend get pods
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods
```

#### 使用 `--show-all` 选项查看

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods --show-all
NAME                                            READY     STATUS        RESTARTS   AGE
anatawa-production-api-v008-6zbuc               1/1       Running       0          9d
anatawa-production-sidekiq-v004-vzebm           1/1       Running       0          120d
...
wechatgo-prod-v019-x7u9o                        1/1       Running       0          7d
[deployer@k8s-deploy-prod0]$
```

> 注意：
>
> - 若不指定 `--show-all` 选项，则 "info: <num> completed object(s) was(were) not shown in pods list. Pass --show-all to see all objects."
> - `kubectl get` command will **hide** resources that have **completed**. For instance, pods that are in the **Succeeded** or **Failed** phases. You can see the full results for any resource by providing the '--show-all' flag.

由此可知 Pods 的 STATUS 会包含如下 phase 类型：

- Running
- Pending
- Completed
- Terminating
- Succeeded     -- 显示为 Completed
- Failed        -- 显示为 Completed


#### 使用 `--show-all` + -`o wide` 选项查看

能够展示更多有用信息：

- IP 列对应 podIP ；
- NODE 列对应 pod 所属节点，即 nodeName 信息；

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods -o wide --show-all
NAME                                                        READY     STATUS        RESTARTS   AGE       IP               NODE
anatawa-production-api-v008-6zbuc                           1/1       Running       0          9d        100.97.53.11     ip-172-1-38-210.cn-north-1.compute.internal
anatawa-production-sidekiq-v004-vzebm                       1/1       Running       0          120d      100.96.253.207   ip-172-1-56-178.cn-north-1.compute.internal
answerup-production-rpc-v020-6f3ej                          1/1       Running       2          30d       100.97.59.6      ip-172-1-43-146.cn-north-1.compute.internal
answerup-production-rpc-v020-fecd6                          1/1       Running       2          29d       100.97.64.21     ip-172-1-45-9.cn-north-1.compute.internal
answerup-production-rpc-v020-hfwsb                          1/1       Running       0          27d       100.97.98.3      ip-172-1-37-106.cn-north-1.compute.internal
answerup-production-rpc-v020-l5gqi                          1/1       Running       3          29d       100.97.75.5      ip-172-1-63-77.cn-north-1.compute.internal
answerup-production-rpc-v020-vhnuv                          1/1       Running       11         40d       100.97.52.211    ip-172-1-53-147.cn-north-1.compute.internal
asher-prod-http-v007-4owut                                  1/1       Running       0          27d       100.97.88.38     ip-172-1-32-224.cn-north-1.compute.internal
asher-prod-http-v007-jx1gm                                  1/1       Running       0          27d       100.96.217.252   ip-172-1-44-48.cn-north-1.compute.internal
asher-prod-sidekiq-v005-6q0fw                               1/1       Running       0          27d       100.97.78.27     ip-172-1-33-129.cn-north-1.compute.internal
asher-prod-sidekiq-v005-c1pxy                               1/1       Running       0          27d       100.97.38.212    ip-172-1-55-121.cn-north-1.compute.internal
audcnv-v002-p7ed1                                           1/1       Running       5          5h        100.97.93.197    ip-172-1-40-92.cn-north-1.compute.internal
audcnv-v002-s5vhw                                           1/1       Running       12         19h       100.97.101.158   ip-172-1-44-230.cn-north-1.compute.internal
checki-3fd1c63108479668                                     0/1       Completed     0          20d       100.97.37.229    ip-172-1-41-215.cn-north-1.compute.internal
checki-3fd1d6b31182ec16                                     0/1       Completed     0          20d       100.97.82.25     ip-172-1-59-228.cn-north-1.compute.internal
checki-3fe19c097c458818                                     0/1       Pending       0          7d        <none>
checki-3fe1a48d658aaa02                                     0/1       Pending       0          7d        <none>
checki-prod-cron-v000-30s38                                 1/1       Running       0          21h       100.97.108.37    ip-172-1-53-68.cn-north-1.compute.internal
checki-prod-rpc-v000-fpprw                                  1/1       Running       0          21h       100.97.65.32     ip-172-1-55-247.cn-north-1.compute.internal
cooper-prod-rpc-v009-2xe7t                                  1/1       Running       0          3d        100.97.98.176    ip-172-1-37-106.cn-north-1.compute.internal
cooper-prod-rpc-v009-ity21                                  1/1       Running       0          3d        100.97.16.228    ip-172-1-57-200.cn-north-1.compute.internal
cooper-prod-rpc-v009-urqr1                                  1/1       Running       0          3d        100.96.184.96    ip-172-1-38-108.cn-north-1.compute.internal
coursecenter-prod-rpc-v005-1satu                            1/1       Running       0          7d        100.97.102.197   ip-172-1-42-238.cn-north-1.compute.internal
coursecenter-prod-rpc-v005-gzgoj                            1/1       Running       0          7d        100.97.84.184    ip-172-1-36-150.cn-north-1.compute.internal
coursecenter-prod-rpc-v005-ivcth                            1/1       Running       0          7d        100.97.72.100    ip-172-1-60-55.cn-north-1.compute.internal
coursecenter-prod-rpc-v005-j4mvv                            1/1       Running       0          7d        100.97.99.98     ip-172-1-52-100.cn-north-1.compute.internal
coursecenter-prod-rpc-v005-zwzy4                            1/1       Running       0          7d        100.97.98.154    ip-172-1-37-106.cn-north-1.compute.internal
...
```

#### 使用 `--show-all` + -`o json` 选项查看

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods -o json --show-all
{
    "kind": "List",
    "apiVersion": "v1",
    "metadata": {},
    "items": [
        {
            "kind": "Pod",
            "apiVersion": "v1",
            "metadata": {
                "name": "-api-v008-6zbuc",
                "generateName": "anatawa-production-api-v008-",
                "namespace": "backend",
                "selfLink": "/api/v1/namespaces/backend/pods/anatawa-production-api-v008-6zbuc",
                "uid": "d0639c40-f9a0-11e7-b481-02ab8baa063e",
                "resourceVersion": "383187508",
                "creationTimestamp": "2018-01-15T03:04:34Z",
                "labels": {
                    "anatawa-production-api": "true",
                    "app": "anatawa",                       -- 所属 app
                    "cluster": "anatawa-production-api",    -- 所属集群
                    "detail": "api",
                    "load-balancer-anatawa-production": "true",
                    "pod-template-hash": "3453961338",
                    "replication-controller": "anatawa-production-api-v008",  -- 对应 rc
                    "stack": "production",
                    "version": "8"
                },
                "annotations": {
                    "kubernetes.io/created-by": "{\"kind\":\"SerializedReference\",\"apiVersion\":\"v1\",\"reference\":{\"kind\":\"ReplicaSet\",\"namespace\":\"backend\",\"name\":\"anatawa-production-api-v008\",\"uid\":\"cfbbc44c-f9a0-11e7-b481-02ab8baa063e\",\"apiVersion\":\"extensions\",\"resourceVersion\":\"381543425\"}}\n"
                },
                "ownerReferences": [
                    {
                        "apiVersion": "extensions/v1beta1",
                        "kind": "ReplicaSet",
                        "name": "anatawa-production-api-v008",
                        "uid": "cfbbc44c-f9a0-11e7-b481-02ab8baa063e",
                        "controller": true
                    }
                ]
            },
            "spec": {
                "volumes": [
                    {
                        "name": "default-token-wdfnf",
                        "secret": {
                            "secretName": "default-token-wdfnf",
                            "defaultMode": 420
                        }
                    }
                ],
                "containers": [
                    {
                        "name": "backend-rls-anatawa",   -- 容器名
                        "image": "prod-reg.llsops.com/backend-rls/anatawa:0.4.0-db1fc2c6",
                        "command": [
                            "unicorn"
                        ],
                        "args": [
                            "-c",
                            "/opt/apps/anatawa/config/unicorn/production.rb"
                        ],
                        "ports": [
                            {
                                "name": "http",
                                "hostPort": 8080,       -- 容器对外暴露的端口
                                "containerPort": 8080,  -- 容易内部使用的端口
                                "protocol": "TCP"
                            }
                        ],
                        "env": [
                            {
                                "name": "SECRET_KEY_BASE",
                                "value": "ab04bf9ae0c9d0da384a3f641ed291a4cd68a782fc5e820ba36f4c3f958b1cb679b8b6723794d8dc6749e9b7a6dc57cb09dfd87f46a2eff8b40c583456193923"
                            },
                            {
                                "name": "MYSQL_HOST",
                                "value": "prod-union.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn"
                            },
                            {
                                "name": "MYSQL_PORT",
                                "value": "6693"
                            },
                            {
                                "name": "MYSQL_USER",
                                "value": "anatawa"
                            },
                            {
                                "name": "MYSQL_PASSWORD",
                                "value": "cribbage.vincible.tamarind.nibs.fanciful.harangue.flossy"
                            },
                            {
                                "name": "NEO_URL",
                                "value": "http://lms.llsapp.com"
                            },
                            {
                                "name": "NEO_CLIENT_TOKEN",
                                "value": "18465bf0a3476fbcce65883d762705af"
                            },
                            {
                                "name": "YUNPIAN_API_KEY",
                                "value": "e71e48159dc143260c93e0ca684aafd1"
                            },
                            {
                                "name": "MENGWANG_USER",
                                "value": "ja0205"
                            },
                            {
                                "name": "MENGWANG_PASSWORD",
                                "value": "985522"
                            },
                            {
                                "name": "LUOSIMAO_API_KEY",
                                "value": "974cbb24d5dff996056740cadcff5ba2"
                            },
                            {
                                "name": "MYSQL_DATABASE",
                                "value": "anatawa_production"
                            },
                            {
                                "name": "REDIS_HOST",
                                "value": "anatawa-sidekiq.qh1egb.0001.cnn1.cache.amazonaws.com.cn"
                            },
                            {
                                "name": "RAILS_ENV",
                                "value": "production"
                            }
                        ],
                        "resources": {
                            "limits": {
                                "cpu": "1700m",
                                "memory": "1700Mi"
                            },
                            "requests": {
                                "cpu": "1700m",
                                "memory": "1700Mi"
                            }
                        },
                        "volumeMounts": [
                            {
                                "name": "default-token-wdfnf",
                                "readOnly": true,
                                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
                            }
                        ],
                        "terminationMessagePath": "/dev/termination-log",
                        "imagePullPolicy": "IfNotPresent"   -- 镜像拉取策略
                    }
                ],
                "restartPolicy": "Always",
                "terminationGracePeriodSeconds": 30,
                "dnsPolicy": "ClusterFirst",
                "serviceAccountName": "default",
                "serviceAccount": "default",
                "nodeName": "ip-172-1-38-210.cn-north-1.compute.internal",  -- 当前 pod 运行的 node 节点
                "securityContext": {},
                "imagePullSecrets": [
                    {
                        "name": "quay"
                    },
                    {
                        "name": "lls-prod"
                    }
                ]
            },
            "status": {
                "phase": "Running",
                "conditions": [
                    {
                        "type": "Initialized",
                        "status": "True",
                        "lastProbeTime": null,
                        "lastTransitionTime": "2018-01-15T03:04:34Z"
                    },
                    {
                        "type": "Ready",
                        "status": "True",
                        "lastProbeTime": null,
                        "lastTransitionTime": "2018-01-15T03:04:47Z"
                    },
                    {
                        "type": "PodScheduled",
                        "status": "True",
                        "lastProbeTime": null,
                        "lastTransitionTime": "2018-01-15T03:04:34Z"
                    }
                ],
                "hostIP": "172.1.38.210",   -- 即当前 pod 所属 node 节点的 nodeIP
                "podIP": "100.97.53.11",    -- 即 podIP
                "startTime": "2018-01-15T03:04:34Z",
                "containerStatuses": [
                    {
                        "name": "backend-rls-anatawa",  -- 容器名
                        "state": {
                            "running": {
                                "startedAt": "2018-01-15T03:04:47Z"
                            }
                        },
                        "lastState": {},
                        "ready": true,
                        "restartCount": 0,
                        "image": "prod-reg.llsops.com/backend-rls/anatawa:0.4.0-db1fc2c6",
                        "imageID": "docker://sha256:ee26a4ab007a2bdb5dc08b3f6d303511362f62a73af561d3bfa86b84d1b9e798",
                        "containerID": "docker://1395e3c4c0303839fcbaeb9ed758e754fe2e32be2056d1669f00a28edbe1a918"
                    }
                ]
            }
        },
        ...
```

> 上述内容主要包括：
> 
> - pod 所属 cluster 和 app
> - pod 中运行的 container 名字，对应的 image 信息，运行命令，映射的端口，环境变量配置，镜像拉取策略
> - pod 当前运行在哪个 node 节点上
> - podIP 和 nodeIP
> - pod 名字字串含义
> ```
 anatawa-production-api-v008-6zbuc
|       |              |    |     |
|<-app->|              |    |     |
|<-------cluster------>|    |     |
|<-replication controller-->|     |
|<-------------pod name---------->|
> ```


### 获取指定 namespace 下指定 pod 信息

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend get -o wide pod anatawa-production-api-v008-6zbuc

[deployer@k8s-deploy-prod0]$ kubectl -n backend get -o json pod anatawa-production-api-v008-6zbuc
```

通过 describe 查看指定 pod 信息

```
[deployer@k8s-deploy-prod0]$ kubectl --namespace backend describe pod anatawa-production-api-v008-6zbuc
Name:		anatawa-production-api-v008-6zbuc
Namespace:	backend
Node:		ip-172-1-38-210.cn-north-1.compute.internal/172.1.38.210  -- 对应 nodeName/nodeIP
Start Time:	Mon, 15 Jan 2018 03:04:34 +0000
Labels:		anatawa-production-api=true
		app=anatawa
		cluster=anatawa-production-api
		detail=api
		load-balancer-anatawa-production=true
		pod-template-hash=3453961338
		replication-controller=anatawa-production-api-v008
		stack=production
		version=8
Status:		Running
IP:		100.97.53.11      -- 即 podIP
Controllers:	ReplicaSet/anatawa-production-api-v008
Containers:
  backend-rls-anatawa:
    Container ID:	docker://1395e3c4c0303839fcbaeb9ed758e754fe2e32be2056d1669f00a28edbe1a918
    Image:		prod-reg.llsops.com/backend-rls/anatawa:0.4.0-db1fc2c6
    Image ID:		docker://sha256:ee26a4ab007a2bdb5dc08b3f6d303511362f62a73af561d3bfa86b84d1b9e798
    Port:		8080/TCP   -- 即 podPort
    Command:
      unicorn
    Args:
      -c
      /opt/apps/anatawa/config/unicorn/production.rb
    Limits:
      cpu:	1700m
      memory:	1700Mi
    Requests:
      cpu:		1700m
      memory:		1700Mi
    State:		Running
      Started:		Mon, 15 Jan 2018 03:04:47 +0000
    Ready:		True
    Restart Count:	0
    Volume Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-wdfnf (ro)
    Environment Variables:
      SECRET_KEY_BASE:		ab04bf9ae0c9d0da384a3f641ed291a4cd68a782fc5e820ba36f4c3f958b1cb679b8b6723794d8dc6749e9b7a6dc57cb09dfd87f46a2eff8b40c583456193923
      MYSQL_HOST:		prod-union.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn
      MYSQL_PORT:		6693
      MYSQL_USER:		anatawa
      MYSQL_PASSWORD:		cribbage.vincible.tamarind.nibs.fanciful.harangue.flossy
      NEO_URL:			http://lms.llsapp.com
      NEO_CLIENT_TOKEN:		18465bf0a3476fbcce65883d762705af
      YUNPIAN_API_KEY:		e71e48159dc143260c93e0ca684aafd1
      MENGWANG_USER:		ja0205
      MENGWANG_PASSWORD:	985522
      LUOSIMAO_API_KEY:		974cbb24d5dff996056740cadcff5ba2
      MYSQL_DATABASE:		anatawa_production
      REDIS_HOST:		anatawa-sidekiq.qh1egb.0001.cnn1.cache.amazonaws.com.cn
      RAILS_ENV:		production
Conditions:
  Type		Status
  Initialized 	True
  Ready 	True
  PodScheduled 	True
Volumes:
  default-token-wdfnf:
    Type:	Secret (a volume populated by a Secret)
    SecretName:	default-token-wdfnf
QoS Class:	Guaranteed
Tolerations:	<none>
No events.
[deployer@k8s-deploy-prod0]$
```

> 和 get 命令的输出完全一致，格式化后更容易查看


## 获取 Services 相关信息

### 查看 default namespace 下的所有 services 信息

```
[deployer@k8s-deploy-prod0]$ kubectl get svc
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   100.64.0.1   <none>        443/TCP   1y
[deployer@k8s-deploy-prod0]$
```

### 查看所有 namespace 下的所有 services 信息

```
[deployer@k8s-deploy-prod0]$ kubectl get svc --all-namespaces
NAMESPACE     NAME                       CLUSTER-IP       EXTERNAL-IP        PORT(S)              AGE
algorithm     acceptor-freetalk-prod     100.69.24.133    <nodes>            443/TCP              13d
algorithm     acceptor-telis-prod        100.67.106.161   <nodes>            8482/TCP             7d
algorithm     barista-chatsrv            100.69.89.241    <none>             50004/TCP            62d
algorithm     barista-dialoguesrv-prod   100.71.172.28    <none>             50002/TCP            172d
algorithm     barista-reportsrv-prod     100.69.201.64    <none>             50001/TCP            172d
algorithm     chatbot0acceptor           100.66.208.72    a055a1789b95b...   8482/TCP             90d
algorithm     chatbotasrprod--lb         100.65.89.253    <none>             8281/TCP             63d
algorithm     essayscorer-telisprod      100.70.184.174   <none>             80/TCP               8d
algorithm     freetalk-asr-prod          100.65.69.223    <none>             8281/TCP             62d
algorithm     lq-lookupsrv-prod          100.70.6.250     <none>             50073/TCP            174d
algorithm     lq-modelsrv-prod           100.68.161.74    <none>             50071/TCP            174d
algorithm     lq-suggsrv-prod            100.65.135.19    <none>             50072/TCP            174d
algorithm     lqdebug-prod-elb           100.69.27.250    a32ada647870e...   80/TCP               154d
algorithm     lqpt-prod-lb               100.64.137.231   <none>             30022/TCP            175d
algorithm     lqtelis-lookupsrv-prod     100.69.218.140   <none>             50073/TCP            40d
algorithm     lqtelisdebug-prod          100.64.6.107     a9f6d832ae3a1...   80/TCP               37d
algorithm     sesame-chatbot-prod        100.64.109.160   a1496aa7bba0e...   443/TCP,8482/TCP     89d
algorithm     sherpa-faq-prod            100.66.182.197   <none>             50051/TCP            38d
algorithm     sherpa-qahub-prod          100.65.222.190   af7d850cad464...   80/TCP,50001/TCP     56d
algorithm     speech-freetalk            100.65.141.217   <none>             8281/TCP             50d
algorithm     telis-processor            100.66.145.206   <none>             50059/TCP            8d
algorithm     telis-telissrv             100.69.160.223   <none>             50005/TCP            8d
backend       anatawa-production         100.65.109.23    <nodes>            80/TCP               132d
backend       answerup-production-lb     100.67.167.150   <none>             50067/TCP            260d
backend       asher-prod-http-lb         100.64.24.19     <nodes>            80/TCP               35d
backend       audcnv                     100.65.81.223    <none>             51236/TCP            114d
backend       checki-prod-rpc            100.71.180.66    <none>             58888/TCP            42d
backend       cooper-prod-rpc-lb         100.68.155.73    internal-a5e0...   50051/TCP            103d
backend       coursecenter-prod-lb       100.69.221.91    <none>             50052/TCP            153d
backend       crm-prod-lb                100.64.203.197   <nodes>            80/TCP               42d
backend       darwin-prod-inner-lb       100.70.58.123    internal-a36f...   80/TCP               173d
backend       fibre-prod-agent           100.67.122.44    <none>             8888/TCP             63d
backend       fibre-prod-http-lb         100.71.171.136   <none>             8888/TCP             62d
backend       fibre-prod-rpc-lb          100.67.56.170    <none>             58887/TCP            62d
backend       gather-prod-lb             100.71.42.82     <none>             50055/TCP            261d
backend       godsaid-prod-lb            100.66.63.252    <nodes>            80/TCP               83d
backend       hexley-prod-lb             100.71.144.163   <none>             50052/TCP            48d
backend       homepage-prod-api-lb       100.71.102.67    internal-a9cc...   80/TCP               1y
backend       ielts                      100.69.141.204   a7ee5350b87e6...   80/TCP,443/TCP       153d
backend       ieltscms                   100.70.183.229   <none>             80/TCP               79d
backend       kensaku-search-grpc        100.67.148.171   <none>             50081/TCP            233d
backend       kensaku-searchweb          100.69.7.69      <nodes>            80/TCP               229d
backend       linkagent-prod-rpc-lb      100.64.86.131    <none>             50051/TCP            181d
backend       linkagent-prod-web-lb      100.67.8.155     internal-aa4e...   80/TCP               181d
backend       llspay-prod-http-lb        100.67.130.189   a995a278b80ca...   443/TCP              162d
backend       llspay-prod-outerhttp      100.71.75.192    a77f139069ea1...   443/TCP              124d
backend       llspay-prod-rpc-lb         100.70.205.110   internal-aaa4...   50066/TCP            246d
backend       neo-prod-ccapi-inner-lb    100.71.52.243    internal-a87d...   80/TCP               280d
backend       neo-prod-ccapi-inner-lb2   100.68.156.255   internal-ae07...   80/TCP               161d
backend       neo-prod-extra-inner-lb    100.66.33.86     internal-a8b4...   80/TCP               280d
backend       neo-prod-lmsapi            100.68.242.215   <nodes>            80/TCP               83d
backend       neo-prod-opsapi-lb         100.67.73.31     <nodes>            80/TCP               271d
backend       neo-prod-payapi-inner-lb   100.69.232.164   internal-a6d5...   80/TCP               215d
backend       neo-prod-web-inner-lb      100.68.22.48     internal-a49e...   80/TCP               316d
backend       perf-prod-cc-rpc           100.64.196.57    <nodes>            51060/TCP            13d
backend       perf-prod-cc-stats         100.69.140.119   <nodes>            51061/TCP            13d
backend       perf-prod-darwin-rpc       100.64.245.32    <nodes>            51080/TCP            13d
backend       perf-prod-darwin-stats     100.69.31.243    <nodes>            51081/TCP            13d
backend       perf-prod-pronco-rpc       100.66.209.138   <nodes>            51070/TCP            13d
backend       phoenix                    100.69.84.180    <nodes>            80/TCP               299d
backend       pronco-prod-lb             100.66.111.238   <none>             50051/TCP            301d
backend       russell-prod               100.66.123.195   <none>             50030/TCP            69d
backend       sequence                   100.65.193.42    <none>             50081/TCP            237d
backend       serah-prod-lb              100.67.37.211    <nodes>            80/TCP               35d
backend       smsgo-prod                 100.64.135.46    <none>             54662/TCP            42d
backend       tape-prod-rpc-lb           100.69.118.163   <none>             50051/TCP            146d
backend       teelog-outer-lb            100.68.32.248    <nodes>            443/TCP              7d
backend       vira-prod-rpc-lb           100.70.176.82    <none>             50056/TCP            28d
backend       viras-prod-api-lb          100.65.155.56    <nodes>            80/TCP               28d
backend       wechatgo-prod              100.67.43.119    internal-ac0c...   8367/TCP,58362/TCP   81d
default       kubernetes                 100.64.0.1       <none>             443/TCP              1y
frontend      cchybridweb-prod-lb        100.66.6.10      <nodes>            80/TCP               49d
frontend      eelmsfe-prod               100.66.178.245   <nodes>            80/TCP               34d
frontend      freetalkweb-prod-lb        100.66.181.209   <nodes>            80/TCP               106d
frontend      lms-prod-lb                100.71.46.193    <nodes>            80/TCP               251d
frontend      lmsmobile-prod-lb          100.69.142.148   <nodes>            80/TCP               167d
frontend      tmsweb-prod-lb             100.71.140.59    a0defb7a5c2e0...   80/TCP,443/TCP       78d
frontend      vira2c-prod                100.67.87.35     <nodes>            80/TCP               27d
frontend      viraadmin-production       100.67.176.199   <nodes>            80/TCP               26d
kube-system   heapster                   100.70.162.208   <none>             80/TCP               1y
kube-system   kube-dns                   100.64.0.10      <none>             53/UDP,53/TCP        1y
kube-system   kubernetes-dashboard       100.65.190.225   <nodes>            80/TCP               1y
kube-system   monitoring-influxdb        100.64.100.168   <none>             8086/TCP             260d
platform      cwexporter-prod-rds        100.67.165.73    <nodes>            80/TCP               139d
platform      dva-server                 100.66.143.181   <nodes>            80/TCP               134d
platform      dva-web                    100.71.10.90     <nodes>            80/TCP               134d
platform      prometheus-service-v2      100.68.147.237   <nodes>            80/TCP               41d
platform      sectoken-prod-rpc          100.65.101.254   <none>             50051/TCP            57d
[deployer@k8s-deploy-prod0]$
```

### 查看指定 namespace 下指定 service 信息

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend get svc anatawa-production
NAME                 CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
anatawa-production   100.65.109.23   <nodes>       80/TCP    133d
[deployer@k8s-deploy-prod0]$ kubectl -n backend get svc anatawa-production -o wide
NAME                 CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE       SELECTOR
anatawa-production   100.65.109.23   <nodes>       80/TCP    133d      load-balancer-anatawa-production=true
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ kubectl -n backend get svc anatawa-production -o json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "anatawa-production",   -- 服务的名字
        "namespace": "backend",
        "selfLink": "/api/v1/namespaces/backend/services/anatawa-production",
        "uid": "89833228-9848-11e7-b481-02ab8baa063e",
        "resourceVersion": "392769791",
        "creationTimestamp": "2017-09-13T05:58:17Z"
    },
    "spec": {
        "ports": [
            {
                "name": "http",
                "protocol": "TCP",
                "port": 80,          -- 服务的 clusterPort
                "targetPort": 8080,  -- 对应 podPort ，即作为 endpoint 端口
                "nodePort": 30605    -- 对外暴露的 NodePort 端口
            }
        ],
        "selector": {
            "load-balancer-anatawa-production": "true"
        },
        "clusterIP": "100.65.109.23",    -- 服务的 clusterIP
        "type": "NodePort",              -- 允许基于 NodePort 模式进行访问
        "sessionAffinity": "None"
    },
    "status": {
        "loadBalancer": {}
    }
}
[deployer@k8s-deploy-prod0]$
```

基于 describe 命令查看

```
[deployer@k8s-deploy-prod0]$ kubectl --namespace backend describe svc anatawa-production
Name:			anatawa-production
Namespace:		backend
Labels:			<none>
Selector:		load-balancer-anatawa-production=true
Type:			NodePort              -- 对外提供服务的方式
IP:			100.65.109.23           -- 服务的 clusterIP
Port:			http	80/TCP      -- 服务的 clusterPort
NodePort:		http	30605/TCP   -- 对外暴露的 NodePort 端口
Endpoints:		100.97.53.11:8080   -- podIP:podPort ，当前 service 下的 EPs
Session Affinity:	None
No events.
[deployer@k8s-deploy-prod0]$
```

从这里可以知道：当访问 `clusterIP:clusterPort` 100.65.109.23:80 时，会被转发到 `podIP:podPort` 100.97.53.11:8080 这个 endpoint 上；


## 获取 Endpoints 相关信息

### 获取 default namespace 下的 endpoints 信息

```
[deployer@k8s-deploy-prod0]$ kubectl get endpoints
NAME         ENDPOINTS          AGE
kubernetes   172.1.55.130:443   1y
[deployer@k8s-deploy-prod0]$
```

### 获取所有 namespace 下的 endpoints 信息

```
[deployer@k8s-deploy-prod0]$ kubectl get endpoints --all-namespaces
NAMESPACE     NAME                       ENDPOINTS                                                                    AGE
algorithm     acceptor-freetalk-prod     100.97.64.139:8482,100.97.82.206:8482                                        14d
algorithm     acceptor-telis-prod        100.97.89.138:8482,100.97.90.205:8482                                        8d
algorithm     barista-chatsrv            100.97.101.3:50004,100.97.70.15:50004                                        63d
algorithm     barista-dialoguesrv-prod   100.97.104.19:50002,100.97.16.133:50002                                      174d
algorithm     barista-reportsrv-prod     100.97.39.16:50001,100.97.53.26:50001                                        174d
algorithm     chatbot0acceptor           <none>                                                                       91d
algorithm     chatbotasrprod--lb         <none>                                                                       64d
algorithm     essayscorer-telisprod      100.97.109.12:50026,100.97.112.56:50026                                      9d
algorithm     freetalk-asr-prod          <none>                                                                       63d
algorithm     lq-lookupsrv-prod          100.97.12.203:50073,100.97.16.207:50073,100.97.59.33:50073                   175d
algorithm     lq-modelsrv-prod           100.96.179.197:50071,100.97.100.5:50071,100.97.39.17:50071 + 2 more...       175d
algorithm     lq-suggsrv-prod            100.97.100.106:50072,100.97.103.128:50072,100.97.104.118:50072 + 9 more...   175d
algorithm     lqdebug-prod-elb           100.97.50.109:8080                                                           155d
algorithm     lqpt-prod-lb               100.97.19.250:30022                                                          176d
algorithm     lqtelis-lookupsrv-prod     100.97.22.180:50073,100.97.39.131:50073                                      41d
algorithm     lqtelisdebug-prod          100.97.112.57:8080                                                           38d
algorithm     sesame-chatbot-prod        100.97.40.38:8482,100.97.93.5:8482,100.97.40.38:8482 + 1 more...             91d
algorithm     sherpa-faq-prod            100.97.61.113:50051,100.97.88.190:50051                                      39d
algorithm     sherpa-qahub-prod          100.97.66.167:80,100.97.73.100:80,100.97.66.167:50001 + 1 more...            57d
algorithm     speech-freetalk            100.97.105.35:8281,100.97.106.24:8281,100.97.47.92:8281 + 1 more...          51d
algorithm     telis-processor            100.97.106.25:50059,100.97.47.93:50059,100.97.50.111:50059 + 1 more...       9d
algorithm     telis-telissrv             100.97.50.112:50005,100.97.92.233:50005                                      9d
backend       anatawa-production         100.97.53.11:8080                                                            134d
backend       answerup-production-lb     100.97.52.211:50067,100.97.59.6:50067,100.97.64.21:50067 + 2 more...         261d
backend       asher-prod-http-lb         100.96.217.252:8080,100.97.88.38:8080                                        36d
backend       audcnv                     100.97.110.34:51236,100.97.41.86:51236                                       115d
backend       checki-prod-rpc            100.97.65.32:59493                                                           43d
backend       cooper-prod-rpc-lb         100.97.110.35:50051,100.97.41.94:50051,100.97.97.175:50051                   104d
backend       coursecenter-prod-lb       100.97.102.197:50052,100.97.72.100:50052,100.97.84.184:50052 + 2 more...     154d
backend       crm-prod-lb                100.97.48.125:8080,100.97.72.3:8080                                          43d
backend       darwin-prod-inner-lb       100.97.65.211:8080,100.97.85.179:8080                                        175d
backend       fibre-prod-agent           <none>                                                                       64d
backend       fibre-prod-http-lb         100.97.101.5:8888,100.97.70.3:8888                                           63d
backend       fibre-prod-rpc-lb          100.97.40.110:58887,100.97.87.13:58887,100.97.93.4:58887                     63d
backend       gather-prod-lb             100.97.101.6:50055,100.97.62.8:50055,100.97.75.3:50055 + 1 more...           262d
backend       godsaid-prod-lb            100.97.86.38:8080,100.97.96.37:8080                                          84d
backend       hexley-prod-lb             100.97.103.150:50052,100.97.108.40:50052,100.97.110.25:50052 + 7 more...     49d
backend       homepage-prod-api-lb       100.96.179.213:8080,100.97.52.26:8080,100.97.61.4:8080                       1y
backend       ielts                      100.97.103.146:80,100.97.62.190:80,100.97.76.6:80 + 3 more...                154d
backend       ieltscms                   100.97.50.120:8080                                                           80d
backend       kensaku-search-grpc        100.97.102.3:50081,100.97.70.4:50081,100.97.99.3:50081                       234d
backend       kensaku-searchweb          100.97.64.22:8000                                                            231d
backend       linkagent-prod-rpc-lb      100.97.52.241:50051,100.97.59.5:50051                                        182d
backend       linkagent-prod-web-lb      100.97.40.85:8080,100.97.74.4:8080,100.97.83.15:8080                         182d
backend       llspay-prod-http-lb        100.97.76.5:8080,100.97.97.159:8080                                          163d
backend       llspay-prod-outerhttp      100.97.16.153:8090,100.97.19.179:8090                                        125d
backend       llspay-prod-rpc-lb         100.97.15.210:50066,100.97.74.49:50066,100.97.95.118:50066                   247d
backend       neo-prod-ccapi-inner-lb    100.97.101.168:8080,100.97.102.218:8080,100.97.107.141:8080 + 17 more...     281d
backend       neo-prod-ccapi-inner-lb2   100.97.101.168:8080,100.97.102.218:8080,100.97.107.141:8080 + 17 more...     162d
backend       neo-prod-extra-inner-lb    100.96.253.226:8080,100.97.108.47:8080,100.97.109.42:8080 + 12 more...       281d
backend       neo-prod-lmsapi            100.97.15.239:8080,100.97.37.191:8080                                        85d
backend       neo-prod-opsapi-lb         100.96.184.107:8080,100.97.40.159:8080                                       272d
backend       neo-prod-payapi-inner-lb   100.97.101.166:8080,100.97.66.184:8080,100.97.87.83:8080 + 1 more...         216d
backend       neo-prod-web-inner-lb      100.97.100.121:8080,100.97.101.167:8080,100.97.102.216:8080 + 47 more...     317d
backend       perf-prod-cc-rpc           100.97.109.16:51060,100.97.48.222:51060                                      14d
backend       perf-prod-cc-stats         100.97.113.17:51061                                                          14d
backend       perf-prod-darwin-rpc       <none>                                                                       14d
backend       perf-prod-darwin-stats     <none>                                                                       14d
backend       perf-prod-pronco-rpc       <none>                                                                       14d
backend       phoenix                    100.97.52.33:80,100.97.60.8:80                                               300d
backend       pronco-prod-lb             100.97.48.151:50051,100.97.66.23:50051                                       302d
backend       pt-grpc-lb                 <none>                                                                       23h
backend       russell-prod               100.97.16.218:8080,100.97.97.115:8080                                        70d
backend       sequence                   100.96.184.138:50081,100.97.82.16:50081                                      239d
backend       serah-prod-lb              100.97.100.9:13888,100.97.103.5:13888,100.97.95.5:13888 + 1 more...          36d
backend       smsgo-prod                 100.97.39.120:54662,100.97.92.232:54662                                      43d
backend       tape-prod-rpc-lb           100.97.15.92:50051,100.97.61.6:50051                                         147d
backend       teelog-outer-lb            100.97.87.36:80                                                              8d
backend       vira-prod-rpc-lb           100.97.104.134:50056,100.97.51.111:50056                                     29d
backend       viras-prod-api-lb          100.97.63.63:8000,100.97.73.23:8000                                          29d
backend       wechatgo-prod              100.97.80.95:58362,100.97.80.95:8367                                         83d
default       kubernetes                 172.1.55.130:443                                                             1y
frontend      cchybridweb-prod-lb        100.97.94.173:8080                                                           50d
frontend      eelmsfe-prod               100.96.179.184:8080                                                          36d
frontend      freetalkweb-prod-lb        100.97.97.4:8080                                                             108d
frontend      lms-prod-lb                100.97.14.240:8080                                                           252d
frontend      lmsmobile-prod-lb          100.97.12.159:8080                                                           168d
frontend      tmsweb-prod-lb             100.97.39.207:8080,100.97.39.207:8080                                        79d
frontend      vira2c-prod                100.97.112.106:8080                                                          28d
frontend      viraadmin-production       100.97.61.9:8080                                                             28d
kube-system   heapster                   100.97.60.10:8082                                                            1y
kube-system   kube-controller-manager    <none>                                                                       1y
kube-system   kube-dns                   100.97.76.4:53,100.97.77.3:53,100.97.78.3:53 + 7 more...                     1y
kube-system   kube-scheduler             <none>                                                                       1y
kube-system   kubernetes-dashboard       100.97.60.4:9090                                                             1y
kube-system   monitoring-influxdb        100.96.184.25:8086                                                           261d
platform      cwexporter-prod-rds        100.97.19.169:9042,100.97.53.30:9042                                         140d
platform      dva-server                 100.97.77.158:8080,100.97.90.215:8080                                        135d
platform      dva-web                    100.97.85.250:8000                                                           135d
platform      prometheus-service-v2      100.96.251.34:9090                                                           42d
platform      sectoken-prod-rpc          100.97.83.139:50051,100.97.88.223:50051                                      58d
[deployer@k8s-deploy-prod0]$
```

### 获取指定 namespace 下的 endpoints 信息

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend get endpoints
NAME                       ENDPOINTS                                                                  AGE
anatawa-production         100.97.53.11:8080                                                          134d
answerup-production-lb     100.97.52.211:50067,100.97.59.6:50067,100.97.64.21:50067 + 2 more...       261d
asher-prod-http-lb         100.96.217.252:8080,100.97.88.38:8080                                      36d
audcnv                     100.97.110.34:51236,100.97.41.86:51236                                     115d
checki-prod-rpc            100.97.65.32:59493                                                         43d
cooper-prod-rpc-lb         100.97.110.35:50051,100.97.41.94:50051,100.97.97.175:50051                 104d
coursecenter-prod-lb       100.97.102.197:50052,100.97.72.100:50052,100.97.84.184:50052 + 2 more...   154d
crm-prod-lb                100.97.48.125:8080,100.97.72.3:8080                                        43d
darwin-prod-inner-lb       100.97.65.211:8080,100.97.85.179:8080                                      175d
...
```

### 获取指定 namespace 下指定 service 中的 endpoints 信息

> anatawa-production 其实是 service name

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend get endpoints anatawa-production -o json
{
    "kind": "Endpoints",
    "apiVersion": "v1",
    "metadata": {
        "name": "anatawa-production",
        "namespace": "backend",
        "selfLink": "/api/v1/namespaces/backend/endpoints/anatawa-production",
        "uid": "89842cc9-9848-11e7-b481-02ab8baa063e",
        "resourceVersion": "392769250",
        "creationTimestamp": "2017-09-13T05:58:17Z"
    },
    "subsets": [
        {
            "addresses": [
                {
                    "ip": "100.97.53.11",  -- 即 podIP
                    "nodeName": "ip-172-1-38-210.cn-north-1.compute.internal",  -- 当前 endpoint 所在 node 名字
                    "targetRef": {
                        "kind": "Pod",
                        "namespace": "backend",
                        "name": "anatawa-production-api-v008-6zbuc", -- 当前 endpoint 关联的 pod
                        "uid": "d0639c40-f9a0-11e7-b481-02ab8baa063e",
                        "resourceVersion": "383187508"
                    }
                }
            ],
            "ports": [
                {
                    "name": "http",
                    "port": 8080,    -- 即 podPort
                    "protocol": "TCP"
                }
            ]
        }
    ]
}
[deployer@k8s-deploy-prod0]$
```

基于 describe 命令查看

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend describe endpoints anatawa-production
Name:		anatawa-production
Namespace:	backend
Labels:		<none>
Subsets:
  Addresses:		100.97.53.11   -- 即 podIP
  NotReadyAddresses:	<none>
  Ports:
    Name	Port	Protocol
    ----	----	--------
    http	8080	TCP           -- 即 podPort

No events.
[deployer@k8s-deploy-prod0]$
```