# mosquitto
- [mosquitto](#mosquitto)
  - [Workshopの内容](#workshop%E3%81%AE%E5%86%85%E5%AE%B9)
  - [Lab1 - Mosquitto-Basic(TLSなし)](#lab1---mosquitto-basictls%E3%81%AA%E3%81%97)
    - [ローカルで実行してみる](#%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%AB%E3%81%A7%E5%AE%9F%E8%A1%8C%E3%81%97%E3%81%A6%E3%81%BF%E3%82%8B)
    - [IKSにデプロイしてみる](#iks%E3%81%AB%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%81%97%E3%81%A6%E3%81%BF%E3%82%8B)
  - [Lab2 - Mosquitto-TLSあり](#lab2---mosquitto-tls%E3%81%82%E3%82%8A)
    - [証明書の作成とConfigの設定](#%E8%A8%BC%E6%98%8E%E6%9B%B8%E3%81%AE%E4%BD%9C%E6%88%90%E3%81%A8config%E3%81%AE%E8%A8%AD%E5%AE%9A)
    - [IKSにデプロイしてみる](#iks%E3%81%AB%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%81%97%E3%81%A6%E3%81%BF%E3%82%8B-1)
  - [Option - NodeRedでMQTTのPub/Sub](#option---nodered%E3%81%A7mqtt%E3%81%AEpubsub)
  - [お掃除](#%E3%81%8A%E6%8E%83%E9%99%A4)
  - [Tops](#tops)
  - [APPENDIX](#appendix)

## Workshopの内容

* やること
  * mosquitto をIKS上にデプロイしてみる

* 必要なもの
  * mqttclient
    * mac: ```brew install mosquitto```

* 利用するもの
  * Docker Image
    * eclipse-mosquitto - Docker Hub　https://hub.docker.com/_/eclipse-mosquitto?tab=description
    * 設定ファイル、Data、Logの場所
      - /mosquitto/config
      - /mosquitto/data
      - /mosquitto/log


## Lab1 - Mosquitto-Basic(TLSなし)
### ローカルで実行してみる
```
# configなし
$ docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

### IKSにデプロイしてみる

1. IKSにアクセス

   * 作成したクラスタの**アクセスタブ**を参照ください。
   * ```kubectl get nodes``` が実行できればOKです。
 
2. deploymentをdeploy
    ```
    kubectl apply -f mosquitto-deployment.yml
    # 結果確認
    kubectl get all --selector='app=mqtt'
    ```

3. serviceを作成
    ```
    # 作成
    kubectl apply -f mosquitto-service.yml

    # サービスのIP、ポート確認
    kubectl get svc mosquitto-basic
    ```

    出力結果例
    ```
    NAME              TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
    mosquitto-basic   LoadBalancer   172.21.149.242   161.202.134.92   1883:30243/TCP   2m39s
    ```


## Lab2 - Mosquitto-TLSあり

   * TLS, Persistent Volume, mosquitto.config を活用した構成
   * 証明書やキーはDocker Imageに入れてしまうとセキュリティリスクとなるため、KubernetesのConfigmapやSecretを利用し実行時に受け渡します。

### 証明書の作成とConfigの設定

1. 証明書の作成

    - 自己認証局
      - ca.crt 認証局の証明書
      - ca.key 認証局の秘密鍵
    - サーバー
      - server.key サーバーの秘密鍵
      - server.crt サーバーの証明書
      - server.csr 自己認証局によって署名されたサーバー証明書


    1. 自己認証局の証明書と秘密鍵の生成
        ```
        openssl req -new -x509 -days 365 -keyout secrets/ca.key -out secrets/ca.crt
        ```

        サンプル入力内容
        ```
        # 検証のためCountry Name、Organization Name、Common Nameは適当に入力ください
        Country Name (2 letter code) []:jp
        State or Province Name (full name) []:tokyo
        Locality Name (eg, city) []:
        Organization Name (eg, company) []:Hoge Inc.
        Organizational Unit Name (eg, section) []:
        Common Name (eg, fully qualified host name) []:mosquitto.default.svc.cluster.local
        Email Address []:
        ```

    2. サーバーの秘密鍵と証明書を発行
        ```
        openssl genrsa -out secrets/server.key 2048
        openssl req -out secrets/server.csr -key secrets/server.key -new
        ```

        サンプル入力内容
        ```
        Country Name (2 letter code) []:jp
        State or Province Name (full name) []:tokyo
        Locality Name (eg, city) []:
        Organization Name (eg, company) []:Hoge Inc.
        Organizational Unit Name (eg, section) []:
        Common Name (eg, fully qualified host name) []:mosquitto.default.svc.cluster.local
        Email Address []:

        Please enter the following 'extra' attributes
        to be sent with your certificate request
        A challenge password []:
        ```

    3. 自己認証局によって署名されたサーバー証明書を発行
        ```
        openssl x509 -req -in secrets/server.csr -CA secrets/ca.crt -CAkey secrets/ca.key -CAcreateserial -out secrets/server.crt -days 365
        ```

### IKSにデプロイしてみる


1. Secret,ConfigMapの登録

    * Secret

        ```
        # 登録
        kubectl create secret generic mosquitto-certifications --from-file=./secrets/ca.crt --from-file=./secrets/server.crt --from-file=./secrets/server.key
        kubectl label  secret mosquitto-certifications app=mqtt

        # 結果確認
        kubectl get secrets
        kubectl describe secret/mosquitto-certifications
        ```

    * ConfigMap

        ```
        # 登録
        kubectl create configmap mosquitto-conf --from-file=mosquitto.conf
        kubectl label  configmap mosquitto-conf app=mqtt
        # 結果確認
        kubectl get configmap
        kubectl describe configmap/mosquitto-conf
        ```

1. SecretやConfigMapを設定したDeployementの適用
   
   Yamlでの設定内容の確認
    ```:mosquitto-deployment.yml
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: mosquitto-tls
      labels:
        app: mqtt
        env: tls-on
    spec:
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: mqtt
            env: tls-on
        spec:
          containers:
            - image: eclipse-mosquitto:latest
              name: mosquitto-tls
              ports:
                - containerPort: 1883
                - containerPort: 9001
                - containerPort: 8883 #TLS
              volumeMounts:
                - name: mosquitto-certifications        # ConfigmapやSecretのファイルをどのパスにマウントするかの設定
                  mountPath: /etc/mosquitto/certs/
                  readOnly: true
                - name: mosquitto-conf
                  mountPath: /mosquitto/config
        volumes:                                        # ConfigmapやsecretはVolumeとして扱う
            - name: mosquitto-certifications
              secret:
                secretName: mosquitto-certifications
            - name: mosquitto-conf
              configMap:
                name: mosquitto-conf
    ```

    yaml の適用
    ```
    kubectl apply -f mosquitto-deployment.yml
    kubectl apply -f mosquitto-service.yml
    ```

    サービス公開しているIPとポートの確認
    ```
    kubectl get svc mosquitto-tls
    ```

## Option - NodeRedでMQTTのPub/Sub

    NodeRedをIKS上にデプロイしてGUIベースでMQTTのPub/Subクライアントを作る

1. IKSにデプロイ
    ```
    # NoderedのDeploy
    kubectl run nodered --image=nodered/node-red-docker --port=1880 --labels="app=nodered"

    # LoadBarancerでのサービス公開
    kubectl expose deployment nodered --port=1880 --type=LoadBalancer --labels="app=nodered"

    # 公開IPとPortの確認
    kubectl get svc nodered
    ```

    出力結果 以下の場合は 161.202.134.91:31924 にてアクセス可能です
    ```
    NAME      TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
    nodered   LoadBalancer   172.21.212.150   161.202.134.91   1880:31924/TCP   67s
    ```

2. Pub/Subのクライアントを作成

    1. 事前にPub/SubのFlowを作成してあるので、以下jsonをNodeRedにて読み込みしてください。（読み込み方法は画面左上のメニュー＞読み込み＞クリップボード）
        ```
        [{"id":"3e6ff90c.6a5106","type":"comment","z":"9ff2e1e1.e86ee","name":"Publish","info":"","x":70,"y":60,"wires":[]},{"id":"747b2dac.584fd4","type":"comment","z":"9ff2e1e1.e86ee","name":"Subscribe","info":"","x":80,"y":280,"wires":[]},{"id":"fd3f4f41.22a2a","type":"mqtt in","z":"9ff2e1e1.e86ee","name":"","topic":"hoge","qos":"2","broker":"fc02b900.a63b38","x":90,"y":340,"wires":[["3e0c179a.42da08"]]},{"id":"72837d4d.7d4594","type":"inject","z":"9ff2e1e1.e86ee","name":"","topic":"","payload":"","payloadType":"date","repeat":"","crontab":"","once":false,"onceDelay":0.1,"x":100,"y":140,"wires":[["c462b3f7.403fd","c5e91ed0.b34f2"]]},{"id":"5663c09b.70b74","type":"debug","z":"9ff2e1e1.e86ee","name":"","active":true,"tosidebar":true,"console":false,"tostatus":false,"complete":"false","x":430,"y":180,"wires":[]},{"id":"c462b3f7.403fd","type":"mqtt out","z":"9ff2e1e1.e86ee","name":"","topic":"hoge","qos":"","retain":"","broker":"fc02b900.a63b38","x":410,"y":140,"wires":[]},{"id":"c5e91ed0.b34f2","type":"template","z":"9ff2e1e1.e86ee","name":"","field":"payload","fieldType":"msg","format":"handlebars","syntax":"mustache","template":"Pub: {{payload}}","output":"str","x":260,"y":180,"wires":[["5663c09b.70b74"]]},{"id":"3e0c179a.42da08","type":"template","z":"9ff2e1e1.e86ee","name":"","field":"payload","fieldType":"msg","format":"handlebars","syntax":"mustache","template":"Sub: {{payload}}","output":"str","x":260,"y":340,"wires":[["688680c3.f69b6"]]},{"id":"688680c3.f69b6","type":"debug","z":"9ff2e1e1.e86ee","name":"","active":true,"tosidebar":true,"console":false,"tostatus":false,"complete":"false","x":430,"y":340,"wires":[]},{"id":"fc02b900.a63b38","type":"mqtt-broker","z":"","name":"","broker":"mosquitto-basic.default.svc.cluster.local","port":"1883","clientid":"","usetls":false,"compatmode":true,"keepalive":"60","cleansession":true,"birthTopic":"","birthQos":"0","birthPayload":"","closeTopic":"","closeQos":"0","closePayload":"","willTopic":"","willQos":"0","willPayload":""}]
        ```

    2. Publish,SubscribeのMQTTBrokerの設定確認

        MQTTノードにて参照するBrokerの設定がされていることを確認

        1. MQTTノードをダブルクリック > サーバーの設定（鉛筆ボタン）
        2. サーバーとポートを指定
            Basicの場合
            * サーバー名: mosquitto-basic.default.svc.cluster.local
            * ポート番号: 1883
              * kube-dnsを利用したアクセスの場合はポートフォワーディングの必要がないため、1883でアクセスできます


            ※TLSの場合は以下の設定が必要です
            * サーバー名: mosquitto-tls.default.svc.cluster.local
            * ポート番号: 8883
            * SSL/TLS接続を使用: ON
              * TLS設定
                * CA証明書: ca.crt
                * サーバ証明書を確認: OFF
  
    3. デプロイ & 動作確認
        
        デプロイ
        * 画面右上のデプロイボタンをおし、Flowをデプロイします。
        
        動作確認
        * timestampノードの左端のボタンを押します。
        * 左のデバッグタブにPub/Subのメッセージが表示されれば成功です！

## お掃除
```
kubectl delete deploy,svc,configmap,secret -l app=mqtt
kubectl delete deploy,svc -l app=nodered
```

## Tops
* mosquitto.configを変更したい
  - デフォルトのConfig: https://raw.githubusercontent.com/eclipse/mosquitto/a0a37d385db4421d7151f1fe969a7b00d4516c24/mosquitto.conf

## APPENDIX
* CentOS6.9環境でMosquitto-TLSの動作確認手順 - Qiita
    - https://qiita.com/akrian/items/b39d0200e2fc543c4347

* MQTT を使用してリアルタイム・データをストリーミング配信する - IBM Developer
    - https://developer.ibm.com/jp/patterns/use-mqtt-stream-real-time-data-2/

* Kubernetes （Azure AKS）上にVerneMQでMQTT over TLSクラスタを構成する - Qiita
    - https://qiita.com/nmatsui/items/0a46a2d2acd9ddfccaa4

* アノテーションを使用した Ingress のカスタマイズ
    - https://cloud.ibm.com/docs/containers/cs_annotations.html#tcp-ports