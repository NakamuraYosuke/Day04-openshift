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

`new-app`した際にDocker Hubから取得したtomcat:latestのコンテナイメージが、自身の環境のレジストリに追加されていることが確認できます。

イメージストリームリソースは、コンテナーイメージのエイリアスであるイメージストリームタグに関連付けられた、特定のコンテナーイメージを命名する設定です。

OpenShift はイメージストリームに対してアプリケーションをビルドします。OpenShift インストーラーは、インストール時にデフォルトでイメージストリームをいくつか生成します。

使用可能なイメージストリームを確認するには、次のようにoc getコマンドを使用します。

```
$ oc get is -n openshift
```

OpenShift は、イメージストリームの変更を検出し、その変更に基づいて措置を講じます。

例えば、nodejs-010-rhel7 イメージでセキュリティーの問題が発生した場合には、イメージリポジトリ内で更新できます。
また、OpenShiftを使用すると、自動的にアプリケーションコードの新規ビルドをトリガーできます。


### S2I と CLI によるアプリケーションのビルド
S2I（Source To Image）を使用してアプリケーションをビルドする場合は、OpenShift CLI を使用することができます。
CLI の oc new-app コマンドを使用して、 S2Iプロセスによりアプリケーションを作成できます。

```
（例）$ oc new-app php~http://my.git.server.com/my-app --name=myapp
```
- このプロセスで使用されるイメージストリームは、チルダ (~) の左側に表示されます。
- チルダの後の URL は、ソースコードの Git リポジトリの場所を示しています。
- アプリケーション名を設定します。

S2I実行時、ソースコードを実行可能なイメージに変換するときに実行される入力パラメーターとトリガーの定義を担うBuildConfig (bc)が生成されます。

アプリケーションを新規作成した後、ビルドのプロセスが開始されます。
oc get buildsコマンドを使用して、アプリケーションビルドの一覧を表示します。

定義済のBuildConfigからアプリケーションをビルドする場合には、`oc start build BuildConfigName`によりコマンドをトリガーします。

### BuildConfigとDeploymentConfigの関係
BuildConfig Podの役割は、OpenShift でイメージを作成しそれを内部コンテナーレジストリーへプッシュすることです。
ソースコードまたはコンテンツの更新には、通常イメージの更新を保証する新規のビルドが必要です。

DeploymentConfig Podの役割は、PodをOpenShift へデプロイすることです。DeploymentConfig Podを実行した結果、内部コンテナーレジストリーにデプロイされたイメージを含むPodが作成されます。
既存の実行中のPod は、DeploymentConfigリソースの設定方法に応じて破棄される可能性があります。

BuildConfig リソースはコンテナーイメージを作成または更新します。
DeploymentConfig はこの新規イメージまたは更新されたイメージイベントに反応し、コンテナーイメージから Pod を作成します。

## 演習
### S2Iを使用したコンテナアプリケーションの作成
developerユーザにクラスタロール権限を与えましょう。
まずはkubeadminでログインします。
```
$ oc login -u kubeadmin -p XXXXX https://api.crc.testing:6443
```

developerユーザに権限を付与します。
```
$ oc adm policy add-cluster-role-to-user cluster-admin developer
```

developerユーザに切り替えます。
```
$ oc login -u developer -p developer
```

この演習問題用に適当なプロジェクトを作成します。
```
$ oc new-project developer-s2i
```

php:7.3のイメージストリームを用いて、
https://github.com/RedHatTraining/DO180-apps
にある`php-helloworld`を`php-helloworld`という名前のPodで作成します。

```
$ oc new-app -i php:7.3 --name=php-helloworld https://github.com/RedHatTraining/DO180-apps --context-dir php-helloworld
```

ビルドが完了しアプリケーションがデプロイされるまで待機します。

oc get podsコマンドによりビルドプロセスが開始されることを確認します。
```
$ oc get pods
NAME                     READY   STATUS    RESTARTS   AGE
php-helloworld-1-build   1/1     Running   0          82s
```

ビルドのログを確認します。
```
$ oc logs --all-containers -f php-helloworld-1-build
Caching blobs under "/var/cache/blobs".
Getting image source signatures
Copying blob sha256:9cb9f809ac39558fd8c0519afd022e9c05b3181ffc5cc90779fa3aabd9284c91
Cloning "https://github.com/RedHatTraining/DO180-apps" ...
Copying blob sha256:c31b969c152308bc3a0263d721151e0970b8db3ab51fa71caecae277c09ce589
	Commit:	f7cd8963ef353d9173c3a21dcccf402f3616840b (Initial commit, including all apps previously in course)
Copying blob sha256:d3f7e3126399d3d04e7c096926d08fd29d330eb9141e0ff9b6fe6b0c2eeddda2
	Author:	Jordi Sola <someth2say@gmail.com>
	Date:	Fri Oct 4 13:04:03 2019 +0200
Copying blob sha256:f8898a923a1e3d6a8b7f5a52d5dba72cecf191ec779ba68c118e6e2f47a5c195
Copying blob sha256:e2f4e29a190055c4b743cde586f5668a77712975171b8babd6f7a2f2f55a0911
Copying config sha256:ab535ec2b8182850589c84bd0d329b10d4ebaf9dfe9867a5018346ab87c1d69c
Writing manifest to image destination
Storing signatures
Generating dockerfile with builder image image-registry.openshift-image-registry.svc:5000/openshift/php@sha256:2fb218e647070eb35b87bb441918243718eb030d6e1f66124aa12ff6a09985e9
STEP 1: FROM image-registry.openshift-image-registry.svc:5000/openshift/php@sha256:2fb218e647070eb35b87bb441918243718eb030d6e1f66124aa12ff6a09985e9
STEP 2: LABEL "io.openshift.build.commit.date"="Fri Oct 4 13:04:03 2019 +0200"       "io.openshift.build.commit.id"="f7cd8963ef353d9173c3a21dcccf402f3616840b"       "io.openshift.build.commit.ref"="master"       "io.openshift.build.commit.message"="Initial commit, including all apps previously in course"       "io.openshift.build.source-location"="https://github.com/RedHatTraining/DO180-apps"       "io.openshift.build.source-context-dir"="php-helloworld"       "io.openshift.build.image"="image-registry.openshift-image-registry.svc:5000/openshift/php@sha256:2fb218e647070eb35b87bb441918243718eb030d6e1f66124aa12ff6a09985e9"       "io.openshift.build.commit.author"="Jordi Sola <someth2say@gmail.com>"
STEP 3: ENV OPENSHIFT_BUILD_NAME="php-helloworld-1"     OPENSHIFT_BUILD_NAMESPACE="developer-s2i"     OPENSHIFT_BUILD_SOURCE="https://github.com/RedHatTraining/DO180-apps"     OPENSHIFT_BUILD_COMMIT="f7cd8963ef353d9173c3a21dcccf402f3616840b"
STEP 4: USER root
STEP 5: COPY upload/src /tmp/src
STEP 6: RUN chown -R 1001:0 /tmp/src
STEP 7: USER 1001
STEP 8: RUN /usr/libexec/s2i/assemble
---> Installing application source...
=> sourcing 20-copy-config.sh ...
---> 15:31:41     Processing additional arbitrary httpd configuration provided by s2i ...
=> sourcing 00-documentroot.conf ...
=> sourcing 50-mpm-tuning.conf ...
=> sourcing 40-ssl-certs.sh ...
STEP 9: CMD /usr/libexec/s2i/run
STEP 10: COMMIT temp.builder.openshift.io/developer-s2i/php-helloworld-1:e07f8155
Getting image source signatures
Copying blob sha256:f4ccdae920401bce0c03654156c31f37daec16f15d9d5b55cbae4de0e13baa00
Copying blob sha256:82480151b3e273522116dd288a88cdb6c79624423b31e465131e131a8bb9d211
Copying blob sha256:dd305fcac638431f09b87f24c0fc6fc9f9edde0fa235ae16d35e0e7b30448ce5
Copying blob sha256:6212faf014802718d0607ca37cc048166b1cfb12797685f37ff7054d071c151a
Copying blob sha256:6e282067f6f49f59d1b870e5ce5f42cd3b94bde09d81fe32d27a2c97a149d308
Copying blob sha256:a8cc35cdf515c284840c0ee5bd6ea0b15df01e596d86bb67c1e64073061e33d3
Copying config sha256:44eb05f49f64f53d0acfe3137afa657ba7b3211969f45a9e3673dd33ee4ee33d
Writing manifest to image destination
Storing signatures
--> 44eb05f49f6
44eb05f49f64f53d0acfe3137afa657ba7b3211969f45a9e3673dd33ee4ee33d

Pushing image image-registry.openshift-image-registry.svc:5000/developer-s2i/php-helloworld:latest ...
Getting image source signatures
Copying blob sha256:a8cc35cdf515c284840c0ee5bd6ea0b15df01e596d86bb67c1e64073061e33d3
Copying blob sha256:f8898a923a1e3d6a8b7f5a52d5dba72cecf191ec779ba68c118e6e2f47a5c195
Copying blob sha256:9cb9f809ac39558fd8c0519afd022e9c05b3181ffc5cc90779fa3aabd9284c91
Copying blob sha256:c31b969c152308bc3a0263d721151e0970b8db3ab51fa71caecae277c09ce589
Copying blob sha256:e2f4e29a190055c4b743cde586f5668a77712975171b8babd6f7a2f2f55a0911
Copying blob sha256:d3f7e3126399d3d04e7c096926d08fd29d330eb9141e0ff9b6fe6b0c2eeddda2
Copying config sha256:44eb05f49f64f53d0acfe3137afa657ba7b3211969f45a9e3673dd33ee4ee33d
Writing manifest to image destination
Storing signatures
Successfully pushed image-registry.openshift-image-registry.svc:5000/developer-s2i/php-helloworld@sha256:09927d672cd3cc977df5da1aa9d6afb1dc54f3a3b5afab86a703312a6465f0bc
Push successful
```

アプリケーションをテストするためのルートを追加します。
```
$ oc expose service php-helloworld --name developer-helloworld
```

作成したルートのドメインを確認します。
```
$　oc get route developer-helloworld
NAME                   HOST/PORT                                             PATH   SERVICES         PORT       TERMINATION   WILDCARD
developer-helloworld   developer-helloworld-developer-s2i.apps-crc.testing          php-helloworld   8080-tcp                 None
```

前の手順で取得したドメインにHTTP GET リクエストを送信して、アプリケー ションをテストします。
```
$ curl -s developer-helloworld-developer-s2i.apps-crc.testing
Hello, World! php version is 7.3.20
```

