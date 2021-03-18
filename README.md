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
ログインしたら、右側のWorkloadsメニューからDeploymentsを選択します。
![](https://raw.githubusercontent.com/NakamuraYosuke/Day04-openshift/main/images/workload.png)

Podが2つになっていることが確認できます。

コンソールからPodを増やしてみましょう。

作成したtomcatのdeploymentを選択します。
![](https://raw.githubusercontent.com/NakamuraYosuke/Day04-openshift/main/images/podscalebefore.png)
Podの円の横にある^をクリックし、Pod数を5にします。

![](https://raw.githubusercontent.com/NakamuraYosuke/Day04-openshift/main/images/podscaleafter.png)

Podの円が青くなれば成功です。

次にターミナルでも確認してみます。
```
$ oc get pod
```
```
NAME                      READY   STATUS    RESTARTS   AGE
tomcat-75b4656685-2ghnh   1/1     Running   0          18m
tomcat-75b4656685-d2764   1/1     Running   0          8m20s
tomcat-75b4656685-k2rfk   1/1     Running   0          8m21s
tomcat-75b4656685-l2vwf   1/1     Running   0          15m
tomcat-75b4656685-tw9xp   1/1     Running   0          8m20s
```
5つのPodが作成されていることが確認できました。

Podの他にも、Deployment、Service、ImageStreamが作成されています。
```
$ oc get deployment
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
tomcat   5/5     5            5           20m

$ oc get service
NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
tomcat   ClusterIP   10.217.5.188   <none>        8080/TCP   20m

$ oc get imagestream
NAME     IMAGE REPOSITORY                                                          TAGS     UPDATED
tomcat   default-route-openshift-image-registry.apps-crc.testing/kubetest/tomcat   latest   20 minutes ago
```

## ImageStream
OpenShiftにはKubernetesに無いImageStreamというオブジェクトがあります。
ImageStreamは内部的にタグに対応するDockerレジストリを管理し、それに追加で機能を提供します。

先ほど生成された、tomcatのイメージストリームの詳細を確認してみます。

```
$ oc describe is tomcat
```
```
Name:			tomcat
Namespace:		kubetest
Created:		24 minutes ago
Labels:			app=tomcat
			app.kubernetes.io/component=tomcat
			app.kubernetes.io/instance=tomcat
Annotations:		openshift.io/generated-by=OpenShiftNewApp
			openshift.io/image.dockerRepositoryCheck=2021-03-18T16:12:52Z
Image Repository:	default-route-openshift-image-registry.apps-crc.testing/kubetest/tomcat
Image Lookup:		local=false
Unique Images:		1
Tags:			1

latest
  tagged from tomcat:latest

  * tomcat@sha256:02f0c6909b28a287a3472421c135fabb1689b3e12b3c5ae588a3f6f5fe910048
      24 minutes ago
```

`new-app`した際にDocker Hubから取得したtomcat:latestのコンテナイメージは、自身の環境のレジストリに追加されていることが確認できます。

