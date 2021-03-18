# Day04-openshift
## OpenShiftの基本的な操作
RHOCP クラスターと対話するには、主に oc コマンドを使用します。
クラスターと対話するには、ほとんどのオペレーションでログインしたユーザーが必要となります。
ログインするための構文は次のとおりです。
```
$ oc login <clusterUrl>
```

### PODのリソース定義構文
RHOCPは、コンテナーを Kubernetes Pod内で実行します。
コンテナーイメージから Pod を作成するために、OpenShift は Pod のリソース定義を必要とします。
JSON または YAML 形式のテ キストファイルとして記述しますが、oc new-app コマンドまたは OpenShift Web コンソール から、デフォルト値を記述したファイルを生成することも可能です。

kubernetesでは、Podを作る際に下記マニフェスト（tomcat-pod.yaml）を作成し、applyしました。

```
apiVersion: v1
kind: Pod
metadata:
  name: tomcat
  labels:
    name: tomcat
spec:
  containers:
    - resources:
      image: tomcat:latest
      name: tomcat
```

```
$ kubectl apply -f tomcat-pod.yaml
```

OpenShiftの場合には、`new-app`を使用します。
```
$ oc new-app --name=tomcat --docker-image=tomcat:latest
```
```
--> Found container image 08efef7 (5 days old) from Docker Hub for "tomcat:latest"

    * An image stream tag will be created as "tomcat:latest" that will track this image

--> Creating resources ...
    imagestream.image.openshift.io "tomcat" created
    deployment.apps "tomcat" created
    service "tomcat" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/tomcat' 
    Run 'oc status' to view your app.
```
Podの状態を見る場合には、
```
$ oc get pod
```
でステータスを確認することができます。

今回はPodを1つ作成しました。
Podを2つに増やしてみます。
```
$ oc scale deployment tomcat --replicas=2
```
ステータスを確認します。
```
$ oc get pod
NAME                      READY   STATUS    RESTARTS   AGE
tomcat-75b4656685-2ghnh   1/1     Running   0          3m19s
tomcat-75b4656685-l2vwf   1/1     Running   0          7s
```
Podが2つになりました。

今度はOpenShift Consoleから見てみましょう。
コンソールを開き、kubeadminでログインしてみましょう。
```
$ crc console
```
