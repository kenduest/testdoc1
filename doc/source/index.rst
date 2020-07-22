.. documentation master file, created by kenduest
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

介紹 Kubernetes 服務探索模式!
=============================

.. toctree::
   :caption: Contents:

一、 前言
^^^^^^^^^^^^^^^

衆所周知，Kubernetes 是透過 pod 來進行應用程式部署的，同時 pod 可以運行於不同的節點或是在不同節點之間遷移，而且還可以按需求調整副本數量。這些動態性的執行動作是用來保證應用在任何時候都可以獲得存取。爲滿足此一關鍵需求，在 K8S 中我們採用服務探索 (Service Discovery) 模式。


二. 服務探索方式
^^^^^^^^^^^^^^^^

2.1. 非微服務方式 
-----------------


.. image:: https://brobridge.com/bdsres/wp-content/uploads/2019/11/1-2048x1153.png

在微服務設計中，應用程式會被分拆成多份元件，各元件負責處理不同的任務，元件之間則透過網路協定(如 HTTP)進行溝通。假設有一個用戶端元件(consumer)需要傳送 HTTP 訊息給服務端(producer)，而服務元件只有一個應用實例，其方案非常簡單：將服務端的連線資訊(如 IP 位址或 DNS 名稱、協定、Port 等)寫進一份配置文件提供給用戶端使用即可。然而，若要達到高可用，一個單一服務通常會存在多份實例。這個時候用戶端就不能只與其中一個實例掛鉤了，萬一這個服務實例被刪除或是改名了，用戶端就失去連線而導致部分服務中斷。

因此，用戶端需要某些方法來探索出哪些服務實體是健全可用的並與之連線。

2.2. 用戶端探索
---------------

.. image:: https://brobridge.com/bdsres/wp-content/uploads/2019/11/2-2048x1153.png

此模式中，用戶端自行測定要連線的服務實例。它透過連線服務註冊表(service registry)元件來做這件事，註冊表會記錄所有運行中的服務及其端點(endpoint)。每當加入一個服務或是某一服務死掉，服務註冊表會自動更新。這裏是用戶端自行負責負載平衡與分攤請求負載到可用的服務去。

2.3. 服務端探索
---------------

此模式中，在服務實例前面會置入一個負載平衡層，用戶端連線到負載平衡器明確定義的 URL，後者負責將服務導向測定的服務後端。

.. image:: https://brobridge.com/bdsres/wp-content/uploads/2019/11/3-1-2048x1155.png


三. Kubernetes 服務元件
^^^^^^^^^^^^^^^^^^^^^^^

K8S 使用服務元件來處理多個連線場景，服務元件通常會直接提供一個靜態 URL 給用戶端來使用。


3.1 應用間通訊(Inter-App Communication)
---------------------------------------

.. image:: https://brobridge.com/bdsres/wp-content/uploads/2019/11/5-2048x1153.png


此一場合中，會有多個 pod 用來寄存應用的某些部份(如認證)，然後讓其他部份可以連線與使用認證元件。如下定義將會建立一個 ReplicaSet 與一個 Service 來滿足上述需求：

.. code-block:: yaml

  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: openid-svc
  spec:
    selector:
      app: openid
    ports:
    - port: 80
      targetPort: 9000
      protocol: TCP
  ---
  apiVersion: apps/v1
  kind: ReplicaSet
  metadata:
    name: openid
    labels:
      app: openid
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: openid
    template:
      metadata:
        labels:
          app: openid
      spec:
        containers:
        - name: oidc
          image: qlik/simple-oidc-provider

這份 YAML 檔案分爲兩個部分: 前面定義了一個 Service 來回應用戶端的請求，任何需要傳送認證請求的用戶端都需要透過 FQDN (openid-svc.default.svc.cluster.local) 與 openid-svc 對談。後面則定義了一個 ReplicaSet 以實際產生執行認證影像檔的 pod 副本。

如下爲此模式的注意要點：

透過挑選 label ，Service 懂得將服務請求導向哪一個 pod 。
在預設上，容器將會監聽 port 9000。但您可以將服務修改爲監聽其他 port (本例是 80)，並將服務請求導向後臺適當的 port。
由於是內部服務的關係(cluster之外無法存取)，它將會被賦予 cluster 的內部 IP。
接着讓我們產生一個 Ubuntu 的 pod 來驗證設定，裏面會安裝 curl 然後向服務發出一個 HTTP 請求：

.. code-block:: shell

 $ kubectl run -i --tty ubuntu --image=ubuntu:18.04 --restart=Never -- bash -il
 (如果看不到提示符號，請按一下 Enter)
 root@ubuntu:/# apt update && apt install curl -y
 -- (設略輸出) --
 root@ubuntu:/# curl openid-svc
 {"error":"invalid_request","error_description":"unrecognized route"}


如我們所預期的從其中一個 openidc pod 取得 JSON 物件，但 curl 用戶端並不知道回應是來自哪一個 pod 的，也不知道後面有多少個 pod 在支撐服務。

首先要搞懂的是，這裏是透過 DNS 的方法來連線服務的。K8S 自帶部署一個 DNS 伺服器，並且在新服務產生時自動加入該筆條目(entry)。事實上，我們是透過 DNS 名稱而不是 cluster IP 與服務溝通的。此外，還可以透過環境變數來找出我們提供了哪些服務。當服務被建立起來後，所有符合標籤(label)的 pod 都會自動賦予服務連線細節對應的環境變數：


.. code-block:: shell

        root@ubuntu:/# env | grep OPENID
        OPENID_SVC_SERVICE_PORT=80
        OPENID_SVC_PORT_80_TCP_PORT=80
        OPENID_SVC_PORT_80_TCP_PROTO=tcp
        OPENID_SVC_PORT=tcp://10.111.58.252:80
        OPENID_SVC_PORT_80_TCP=tcp://10.111.58.252:80
        OPENID_SVC_SERVICE_HOST=10.111.58.252
        OPENID_SVC_PORT_80_TCP_ADDR=10.111.58.252

這裏我們在一個臨時的 Ubuntu 容器中透過 env 命令獲得以 OPENID_SVC 開頭的環境變數，我們可以看到關於服務的不同細節，例如服務 IP、port、與協定，等等。

使用環境變數的缺點是，服務必須要比 pod 更早被建立，如果您是在其子 pod 已經執行起來之後才建立服務，那將無法將環境變數注入運行中的 pod 。服務的應用非常多樣化，您可以當作一站式工坊來提供負載平衡與服務註冊：

可提供多個 port。例如以 80 port 提供非加密流量再以 443 提供 SSL 加密連線，都可以在同一個服務中揭露。
可提供內部連線(session)的親和性。使用 .spec.session.Affinity: Client IP 可以保證來自特定 pod IP 的流量只會被導向到同一個目標 pod 而不是隨機選取的 pod 。要注意的是，這裏所談的連線親和性是工作在網路第四層的，例如，不能用於 HTTP cookie (這層次的親和性可考慮使用 Ingress)。

進階健康探測。所有的負載平衡器都必須要檢查後端的節點是健康的且能夠迴應請求，如此才能將請求導向給他們。服務利用 K8S 提供的健康及就緒探測來確保 pod 不僅處於運行狀態，而且還返回預期的迴應。
固定 IP。可透過修改 spec.ClusterIP 參數來手動選擇 IP，某些老舊的應用程式可能設定爲連線特定 IP 位址，這時候就很好用了。

3.2 連線外部資源 (Connecting To an Outside Resource)
----------------------------------------------------

.. image:: https://brobridge.com/bdsres/wp-content/uploads/2019/11/6-2048x1168.png

預設上，K8S 服務的運作是透過動態追蹤 pod 得到其對應建立的端點(endpoint)。舉個例子，您配置了一個服務資源去追蹤帶有 app=web 標籤的 pod，然後任何健全的帶相同標籤的 pod 都會從服務接收到連線流量。然而，有時候您或許需要 pod 去連線外部資源，比方說要使用 cluster 外面的第三方 API 服務，您仍可使用服務界面。在此例中，您可以建立一個不帶選擇器的服務並且手動加入端點：

::

 --- apiVersion: v1
 kind: Service
 metadata:
   name: external-ip
 spec:
   ports:
   - port: 80
     protocol: TCP

上面的定義跟我們第一個服務例子非常類似，只是將選擇器的部分移除掉而已。再來，服務需要端點才能運作，因此我們可以使用下面這個定義：

.. code-block:: yaml

   apiVersion: v1
   kind: Endpoints
   metadata:
     name: external-api
   subsets:
     - addresses:
       - ip: 123.123.123.123
       - ip: 134.134.134.134
       ports:
       - port: 80


這裏最重要的事情是端點資源的名稱必須要與服務名稱一致，否則將無法將兩者關聯起來。

或許您會好奇：爲什麼說在 cluster 內的 pod 與外部資源之間加入一個服務層會是一個聰明的做法呢？爲何不直接在 pod 的組態中提供 DNS 名稱或是 IP 位置給外部資源就好呢？其理由如下：

- 一個服務可以有多個 IP 端點。如此的話，若外部 API 因高可用而揭露出多個 IP 位址的話，您可以利用服務於其間進行負載平衡。

- 服務在 pod 與其連線目標之間扮演了一個抽象的層階，您可以在改變目標的同時又不至於影響到用戶端 pod 的組態。比方說遠端伺服器的 IP 位址變更了，您只需要修改一個地方即可。您甚至可以將外部服務交由 cluster 中一個或多個 pod 來處理。從一開始就帶入服務將可容許您透過加入必須的選擇器來引導服務流量到新的 pod 上面去。所有這些修改只需在服務上進行即可，而無須變更 pod 的組態。

我們不再需要花力氣以其他方式建立服務來連線外部目標，用這個方法我們只需要提供外部資源的 DNS 名稱即可，也不需要手動建立端點。此類服務(稱作外部名稱)只要建好 DNS CNAME 記錄就行，而不是透過連線代理(proxy)。下例的定義就是爲 api.example.com 建立 CNAME 即可：

.. code-block:: yaml

   apiVersion: v1
   kind: Service
   metadata:
     name: externalname
   spec:
     type: ExternalName
     externalName: api.example.com
     ports:
     - port: 80

3.3 受理外部連線(Accepting Outside Connections)
-----------------------------------------------

很多時候，您需要受理來自 cluster 外部的連線。假如您是提供網站應用主機服務的，您或許需要用戶端能夠在他們自己的電腦上消費您的服務。然則，有兩種方式可以透過服務將 pod 揭露給外面的世界。


3.3.1 使用 NodePort
>>>>>>>>>>>>>>>>>>>

.. image:: https://brobridge.com/bdsres/wp-content/uploads/2019/11/7-2048x1153.png


此方法就是用 cluster 中任何一臺主機的外部 IP 位址再結合特定的 port。這個 port 必須在所有主機上獲得保留，當然，您可以手動挑選可用的 port (但必須是所有主機都可用才行)，或是讓 Kubernetes 幫您隨機挑選也行(刪掉 nodePort 參數即可)。當流量到達任何一臺主機所分配的 port 都會被服務自動導流到適當的 pod 去。舉個例子，使用下面的定義我們可以使用 NodePort 服務將驗證服務揭露給全世界：


.. code-block:: yaml

   apiVersion: v1
   kind: Service 
   metadata: 
     name: oidc-svc
     spec:
       type: NodePort 
       selector:
         app: openid 
         ports:
         - port: 80
           targetPort: 9000
           nodePort: 30030
           protocol: TCP

在使用 NodePort 方式的時候，請留意如下要點：

- 要是使用內網節點 port 的話，請配置好必要的網路與防火牆規則以允許外部流量。在複雜的架構環境中這有一定的難度。 用戶端應用必須能感知 cluster 中的所有節點，若其中一個節點掛掉，用戶端要能知道並且將連線引導到其他健全的節點。這會多了一層額外的開銷，因爲所有的用戶端服務都必須修改其組態。比較可行的解套則是在節點之前部署一個負載平衡器自動將流量導引到健全的節點。

-當流量抵達隨機節點時，倒不一定就是部署目標 pod 的節點，服務會處理流量並導向帶有目標 pod 的節點。但這也同時多產生了一個不必要的網路跳站所引起的延遲。可行的解套是透過 DaemonSet 確保所有的服務 pod 出現在所有節點之上。另外一個解套是在服務定義中使用 .spec.externalTrafficPolicy: Local，此參數可以確保只有執行目標 pod 的節點才會接收到流量。再次提醒，這裏帶出的負擔是要修改用戶端實體，也就是哪個節點運行哪個 pod。此類手法其實並不值得鼓勵，因爲這更緊密性偶合化用戶端及其目標 pod ，從而繞開了服務存在的真正原因。

- 此外，如果一個節點收到的流量本來是給另外節點的 nod 的，封包的來源位址將會變更爲節點的 IP。然則， 當目標節點收到封包時，就好像它是從 cluster 其他節點送出來的。

3.3.3 使用 LoadBalancer
>>>>>>>>>>>>>>>>>>>>>>>

.. image: https://brobridge.com/bdsres/wp-content/uploads/2019/11/8-2048x1157.png

爲了應付 NodePort 方式的不足，K8S 提供了 LoadBalancer 的方式。使用這一類別時，服務會自動啓動並配置負載平衡器並將流量分散到節點去。

要使用這類服務，您必須將資訊基礎託管給那些支援負載平衡器與 K8S 的雲端服務商。因爲負載平衡器元件是由雲端服務商供應與管理的，因此不同的服務商會有不同的組態設定。例如，您有可能被允許也有可能不被允許選擇負載平衡器所揭露的外部 IP 位址。並且，有些服務商會保留請求端的來源 IP 位置，有些則會被替換爲負載平衡器的 IP 位址。下面的這個定義示範了怎樣建立一個負載平衡器的服務：

.. code-block:: yaml

   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: openid-svc
   spec:
     type: LoadBalancer
     clusterIP: 10.0.150.240
     loadBalancerIP: 80.15.25.20 # Depends on whether the cloud provider allows it
     selector:
       app: openid
     ports:
     - port: 80
       targetPort: 9000
       protocol: TCP


這裏主要修改的地方是 type 被設定爲 LoadBalancer。在此模式中，服務揭露了一個 cluster 內部 IP(Cluster IP)，同時也揭露了一個外部 IP 並透過負載平衡器受理外部流量。


3.4 無頭服務(The Headless Service)
----------------------------------

.. image: https://brobridge.com/bdsres/wp-content/uploads/2019/11/9-2048x1162.png


服務資源之所以存在的主要理由就是能提供一個統一的 URL(或 IP)給用戶端 pod 來連線其他 pod 。傳統上，只要是用戶端所期待的迴應就無需理會這個迴應實際上是來自哪裏的，然而對有狀態的應用來說並非如此。在一個有狀態的應用中，用戶端更關注於連線特定的 pod (例如 cluster 的主節點、ZooKeeper 的領導節點，等)。K8S 有狀態應用都是透過 StatefulSet 來處理的，這樣的話，服務並不是負載平衡分散到 pod 上的，如果服務不能分散流量那還有什麼好處嗎？好吧，我們還是需要服務元件來更新相配 pod 的端點清單啦。這類服務則被稱爲”無頭服務”，在無頭服務定義中，將 clusterIP 參數設定爲 none 就能有效地阻絕了服務揭露其 IP 位址。當服務 DNS 名稱被查詢的時候，服務節點並不迴應 IP 位址，而是返回它管理的 pod 端點清單。然後是用戶端負責挑選所要連線的 pod 。請注意，如果您想要建立 StatefulSet 的話，你就需要建立無頭服務。如下定義示範了無頭服務：


.. code-block:: yaml

   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: openid-svc
   spec:
     selector:
       app: openid
     clusterIP: none
     ports:
     - port: 80
       targetPort: 9000
       protocol: TCP


3.5 入口控制器(Ingress Controller)
----------------------------------

.. image: https://brobridge.com/bdsres/wp-content/uploads/2019/11/%E6%88%AA%E5%9C%96-2019-11-13-%E4%B8%8B%E5%8D%884.29.23-2048x1149.png


如果您的應用只需要一個入口點來提供給用戶端使用，基本上用負載平衡器就行了。然而，時至今日許多應用都揭露出多個端點。例如，我們用 www.example.com 來展覽公司的業務、紀念品與最新優惠等等，但我們同時又有 api.example.com 給別人程式化連線我們的服務，又有 app.example.com 提供圖形版的應用程式，以及 blog.example.com 作爲社羣入口，林林種種的網站。在 K8S 中各網站都是各自由多個 pod 組成的服務，如果您是用平衡負載器來提供每一個服務的話，我們就必須提供 4 個平衡負載器才行。更好的做法是以單一的主入口來提供所有服務就好，將不同的 URL 請求導流到相對的後端服務。然則，我們需要的就是 Ingress 服務了。

不過，Ingress 其實並不是另一種服務類別，而是獨立的 K8S 資源，它有自己的定義與屬性，需要先有 Ingress 控制器才能在 cluster 上運行。關於如何在 cluster 上部署 Ingress 控制器的教學請參考 https://kubernetes.github.io/ingress-nginx/deploy/ 。

Ingress 可以作爲應用的入口點，它位於服務之前並引導流量到不同的服務。但它跟負載平衡器完全不一樣！利用 Ingress 我們可以做到：

- SSL 終結站(作爲 SSL 的終結點，您只需要在 Ingress 上安裝一份憑證即可)
- 進階路由與重寫規則。事實上，Ingress 就是一個反向代理，你可以撰寫客製化規則將 URL 引導到相應的服務。

如下的定義示範了如何用一個單一的 Ingress IP 來引導流量到不同的後端服務：

.. code-block:: yaml

   apiVersion: extensions/v1beta
   kind: Ingress
   metadata:
     name: myoid
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: / 
     spec:
       rules:
       - http:
         paths:
         - path: /
           backend:
             serviceName: openid-svc
             servicePort: 9000
         - path: /health
           backend:
             serviceName: health-svc
             servicePort: 80

其中最重要的設定行如下：

- .metadata.annotations (第5、6行)：Ingress 是透過控制器執行的，控制器所需的額外組態設定則透過 annotation 來傳遞，本例所用的就是改寫(rewrite)。更多複雜的改寫範例可參考： https://kubernetes.github.io/ingress-nginx/examples/rewrite/ 。

在 spec 的部分則是規則列表，每一道規則包含了路徑(也就是捕獲的請求)與後端。後端則指定了哪一個服務來迴應哪一個 URL 請求，以及所監聽的 port 。
- 在實務上，以上的定義指派了一個與 DNS 記錄(如 oid.example.com )關聯的外部 IP 位址。現在，任何時候使用者點擊 app.example.com 都可以得到 openid-svc 服務的迴應；如果點 app.example.com/health 的話實際上會是跟 health-svc 服務對話，得到的迴應將會是應用健康狀態的 JSON 物件。

此一方式可以將服務實作與外部資源解偶，服務定義只需集中在如何引導流量到 pod、如何挑選正確的 pod、以及在跑什麼 port 上面，然後將外部存取與路由規則留給 Ingress 來處理就好。

四、結論
^^^^^^^^^^^^^^^

服務探索是 Kubernetes 核心概念之一，是不同元件間彼此連接的黏着劑。服務資源原本的出現是因應動態變更的 pod 之間的溝通需要統一的穩定界面，然而，當應用變得越來越複雜的時候，服務的使用就不再只有那樣而已。


原文作者：Mohamed Ahmed

感謝Mohamed Ahmed 所借用分享的文章，原文出處：`https://www.magalix.com/blog/kubernetes-patterns-the-service-discovery-pattern`-

