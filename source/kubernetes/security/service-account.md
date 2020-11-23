# ServiceAccount には最小権限を与える

ServiceAccount のトークンと証明書は Pod 内の `/var/run/secrets/kubernetes.io/serviceaccounts/` 配下にマウントされます。  
そのため、Pod が侵害された場合には、攻撃者はマウントされた ServiceAccount の権限でリソースの操作が可能になります。ですので、ServiceAccount への権限付与は最小権限の原則に則り、必要な権限のみを付与することを推奨します。

Pod にマウントする ServiceAccount を明示していない場合は、 `default` ServiceAccount のトークンがマウントされますが、権限が付与されていないため、ほぼ何もできません。

```shell
bash-5.0# cd /var/run/secrets/kubernetes.io/serviceaccount/ bash-5.0# ls
ca.crt     namespace  token

bash-5.0# KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
bash-5.0# curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
  https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT/api/v1/namespaces/default/pods/$HOSTNAME
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods \"test\" is forbidden: User \"system:serviceaccount:lab:default\" cannot get resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "name": "test",
    "kind": "pods"
  },
  "code": 403
}
```

自身が利用可能なアクションを知る API を利用して Pod を `get` できないことが確認できます。

```shell
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" \
     -d @- \
     -H "Content-Type: application/json" \
     -H 'Accept: application/json, */*' \
     -XPOST https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT/apis/authorization.k8s.io/v1/selfsubjectaccessreviews <<'EOF'
{
   "kind":"SelfSubjectAccessReview",
   "apiVersion":"authorization.k8s.io/v1",
   "metadata":{
      "creationTimestamp":null
   },
   "spec":{
      "resourceAttributes":{
         "namespace":"lab",
         "verb":"get",
         "resource":"pods"
      }
   },
   "status":{
   }
}
EOF

{
  "kind": "SelfSubjectAccessReview",
  "apiVersion": "authorization.k8s.io/v1",
  "metadata": {
    "creationTimestamp": null,
    "managedFields": [
      {
        "manager": "curl",
        "operation": "Update",
        "apiVersion": "authorization.k8s.io/v1",
        "time": "2020-11-23T02:39:02Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {"f:spec":{"f:resourceAttributes":{".":{},"f:namespace":{},"f:resource":{},"f:verb":{}}}}
      }
    ]
  },
  "spec": {
    "resourceAttributes": {
      "namespace": "lab",
      "verb": "get",
      "resource": "pods"
    }
  },
  "status": {
    "allowed": false
  }
}
```

もし次のように Job を作成するできるような権限を持った ServiceAccount のトークンがマウントされている場合は、Job を通して Pod を作成することができます。  

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: runner
  namespace: lab

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: job-runner
  namespace: lab
rules:
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs", "job/status"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["pods", "pods/binding", "pods/log", "pods/status"]
    verbs: ["get", "list"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: job-runner
  namespace: lab
subjects:
- kind: ServiceAccount
  name: runner
  namespace: lab
roleRef:
  kind: Role
  name: job-runner
  apiGroup: rbac.authorization.k8s.io
```

```shell
bash-5.0# curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" -H "Content-Type: application/json" -H 'Accept: application/json, */*' -d @- https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT/apis/batch/v1/namespaces/lab/jobs <<'EOF'
{
   "apiVersion":"batch/v1",
   "kind":"Job",
   "metadata":{
      "name":"sleep-job",
      "namespace":"lab"
   },
   "spec":{
      "backoffLimit":4,
      "template":{
         "spec":{
            "containers":[
               {
                  "command":[
                    "sleep",
                    "100"
                  ],
                  "image":"alpine:latest",
                  "name":"sleep-job"
               }
            ],
            "restartPolicy":"Never"
         }
      }
   }
}
EOF
```

例えばもし、 hostPath のマウントを制限していない場合、これを利用して Pod が配置された node にエスケープすることができます。  

## `automountServiceAccountToken` を利用してトークンをマウントしない

Pod に ServiceAccount のトークンをマウントする必要がない場合は `automountServiceAccountToken: false` を指定します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  serviceAccount: runner
  automountServiceAccountToken: false
  ...
```

また、 ServiceAccount に対して `automountServiceAccountToken` を指定することもできます。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: runner
  namespace: lab
automountServiceAccountToken: false
```

ServiceAccount に対して指定した場合、明示的に `automountServiceAccountToken: true` を指定しなければマウントされません。  

## TokenRequestProjection を利用する

TokenRequestProjection を利用することで ServiceAccount トークンを動的に発行して Pod にマウントすることができます。[^1]  
これにより ServiceAccount のトークンの有効期限を設定しつつ、自動で Pod 内のトークンをリフレッシュすることができます。  
また、Pod を削除するとそのトークンも利用不可となるため、トークンが漏洩した場合に影響を小さくすることができます。

例えば次のようなマニフェストを実行すると、10分でリフレッシュされる ServiceAccount トークンをマウントする Pod を作成することができます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sleep
spec:
  serviceAccount: runner
  containers:
  - name: alpine
    image: alpine:latest
    args:
    - tail
    - -f
    - /dev/null
    volumeMounts:
        - mountPath: /var/run/secrets/tokens
          name: token
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 600
          audience: api
```

Pod を実行し、10分経過すると token がリフレッシュされていることが確認できます。

```shell
/run/secrets/tokens # date
Mon Nov 23 08:17:14 UTC 2020
/run/secrets/tokens # cat token
eyJhbGciOiJSUzI1NiIsImtpZCI6IkRqWFZUR3dMZ2tsbXZyUHVGZ01nRHc5d2Q3U3laRjZVRXFHTzQ5eHZaQjAifQ.eyJhdWQiOlsiYXBpIl0sImV4cCI6MTYwNjExOTcxOCwiaWF0IjoxNjA2MTE5MTE4LCJpc3MiOiJrdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJsYWIiLCJwb2QiOnsibmFtZSI6InNsZWVwIiwidWlkIjoiZDQ4N2M5ODQtNzMxNC00MWZmLThjZTUtNWUxNGIyZDc0OGFmIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJydW5uZXIiLCJ1aWQiOiIyYTNjYjZiZi1lOGE5LTRhNDAtYWFlMC1lODMyMjMwNjI5MzIifX0sIm5iZiI6MTYwNjExOTExOCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmxhYjpydW5uZXIifQ.Sr6ZkbaoFlX4QcUO53gloBjkT_hqKYg1wh13qS6lAX1INUi7tVEYWCjKw3RvkocNIeFIa7WWzlgD66vdXT2OV63yd2Zxovndyx68_PSqbYlhluASTiOasT24JGqqN7iq2uwp8hrw5YTjyEenLQhAJ1qC1Xzgh5NQYxcLYErk2NQVFKQzbhrHVZvtl0NlW3lyNmp6beCy1_jZqccyOTWK8p_D0HXRGkSHo1ExYRYqtbIg-f6j61-NwWU0duUbI_i-vRFO7KefW4onv2RBRiOun91by_xCziAYXWch6SFYWSIbxaFvk-jb6OixtMgUI8q514AWb2SGoWQ0xBvAQhikhg

/run/secrets/tokens # date
Mon Nov 23 08:29:03 UTC 2020
/run/secrets/tokens # cat token
eyJhbGciOiJSUzI1NiIsImtpZCI6IkRqWFZUR3dMZ2tsbXZyUHVGZ01nRHc5d2Q3U3laRjZVRXFHTzQ5eHZaQjAifQ.eyJhdWQiOlsiYXBpIl0sImV4cCI6MTYwNjEyMDIwNCwiaWF0IjoxNjA2MTE5NjA0LCJpc3MiOiJrdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJsYWIiLCJwb2QiOnsibmFtZSI6InNsZWVwIiwidWlkIjoiZDQ4N2M5ODQtNzMxNC00MWZmLThjZTUtNWUxNGIyZDc0OGFmIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJydW5uZXIiLCJ1aWQiOiIyYTNjYjZiZi1lOGE5LTRhNDAtYWFlMC1lODMyMjMwNjI5MzIifX0sIm5iZiI6MTYwNjExOTYwNCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmxhYjpydW5uZXIifQ.SgObuy7ql-kXI-P6uNY6hmUdONSZJfPo7dvxukU7kKFCCIvQcNnWYxzOoo2B_XK4_u7atAGtqWSe9MBG6rJpT73lOjSmGMOeqGVKAe6UTpbnbmS9DO6sVnwCNOCRgs_muwTyF6km66ZxvAm866V5kUIoX407Aa5I-KWZk-8OKT9Db6QKgKBqA9lPKX_Ii-AYBVi_kKB1wR70zxNW_VOapMh9oGXU-ymzGDfJb0Cdo8wJJabpgbIWVlEO7E9417gf6w90U_H5b4mOdGsjWs0JtgVXw3sGBflHUrU0AwYUXI6a8B_HFbS4Q0ChYMZCm5amFQvC6lZL5OsILnaG9JwILg
```

古いトークンは利用できなくなっていることも確認しましょう。

```shell
bash-5.0# KUBE_TOKEN=eyJhbGciOiJSUzI1NiIsImtpZCI6IkRqWFZUR3dMZ2tsbXZyUHVGZ01nRHc5d2Q3U3laRjZVRXFHTzQ5eHZaQjAifQ.eyJhdWQiOlsiYXBpIl0sImV4cCI6MTYwNjExOTcxOCwiaWF0IjoxNjA2MTE5MTE4LCJpc3MiOiJrdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJsYWIiLCJwb2QiOnsibmFtZSI6InNsZWVwIiwidWlkIjoiZDQ4N2M5ODQtNzMxNC00MWZmLThjZTUtNWUxNGIyZDc0OGFmIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJydW5uZXIiLCJ1aWQiOiIyYTNjYjZiZi1lOGE5LTRhNDAtYWFlMC1lODMyMjMwNjI5MzIifX0sIm5iZiI6MTYwNjExOTExOCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmxhYjpydW5uZXIifQ.Sr6ZkbaoFlX4QcUO53gloBjkT_hqKYg1wh13qS6lAX1INUi7tVEYWCjKw3RvkocNIeFIa7WWzlgD66vdXT2OV63yd2Zxovndyx68_PSqbYlhluASTiOasT24JGqqN7iq2uwp8hrw5YTjyEenLQhAJ1qC1Xzgh5NQYxcLYErk2NQVFKQzbhrHVZvtl0NlW3lyNmp6beCy1_jZqccyOTWK8p_D0HXRGkSHo1ExYRYqtbIg-f6j61-NwWU0duUbI_i-vRFO7KefW4onv2RBRiOun91by_xCziAYXWch6SFYWSIbxaFvk-jb6OixtMgUI8q514AWb2SGoWQ0xBvAQhikhg

bash-5.0# curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT/api/v1/namespaces/lab/pods/$HOSTNAME
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

Pod を削除すると有効だったトークンも利用できなくなっています。

```shell
$ kubectl delete -f test.pod

bash-5.0# curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT/api/v1/namespaces/lab/pods/$HOSTNAME
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```



---

[^1]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection
