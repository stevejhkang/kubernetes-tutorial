# kubernetes-tutorial
이 레포지토리는 kubespray를 이용한 kubernetes설치방법과 [weaveworks의 양말 쇼핑몰](https://microservices-demo.github.io/) Microservice와 이를 모니터링할 수 있는 [prometheus](https://prometheus.io/)와 [grafana](https://grafana.com/)를 올려봄으로써 kubernetes의 전반적인 내용을 공부할 수 있는 튜토리얼이다.

양말 쇼핑몰 서비스를 설치하면서 pod, deployment, service에 대한 개념을 다룰 예정이며, 모니터링 시스템을 설치하면서 volume, configmap, secret, 권한설정(RBAC), horizontal pod autoscaler(hpa)에 대한 개념을 다루게 된다. 이 레포지토리에서는 쿠버네티스의 개념보다는 쿠버네티스 위에서 돌아가는 오브젝트들을 중점적으로 다룬다. 

```bash
cd $HOME
git clone https://github.com/stevejhkang/kubernetes-tutorial.git
```
  ## kubespray로 쿠버네티스 설치
쿠버네티스에 대한 설명을 하기 전에 [쿠버네티스 설치](<https://github.com/stevejhkang/kubespray>)를 먼저 시켜놓고 설명을 하도록 한다. 쿠버네티스 설치 레포지토리는 [kubernetes-sigs/kubespray](https://github.com/kubernetes-sigs/kubespray)를 fork해서 한글화한 레포지토리이다.

  ## 쿠버네티스란 무엇인가?
설치를 시켜놓으면 30분정도의 시간이 소요되므로 쿠버네티스에 대한 개념과 컴포넌트에 대해 생소하다면 쿠버네티스가 무엇인지, 쿠버네티스 컴포넌트가 무엇인지를 아래 문서를 참조하자.

참조: https://kubernetes.io/ko/docs/concepts/overview/what-is-kubernetes/

  ## 쿠버네티스 컴포넌트
쿠버네티스 클러스터를 제공하기 위해 필요한 바이너리 컴포넌트들로서 마스터에서 돌아가는 컴포넌트와 노드들에서 돌아가는 컴포넌트로 나눌 수 있다.

참조: https://kubernetes.io/ko/docs/concepts/overview/components/

  ## 순서 

  [0. Namespace](#namespace)  
  [1. Pod와 Deployment](#pod와-deployment)    
  [2. Service](#service)   
  [3. Volume](#volume)   
  [4. Configmap](#configmap)   
  [5. 권한설정-RBAC](#권한설정-rbac)   
  [6. 오토스케일링](#오토스케일링)   
  [7. Secret](#secret)  

---
  ### Namespace
쿠버네티스의 namespace는 쿠버네티스 내의 관련된 오브젝트들을 묶음 지을 때 사용한다. 또 동일한 클러스터 내에 다른 namespace안에 동일한 오브젝트를 올리는 것을 가능하게 해준다. 기본으로 있는 namespace는 default, kube-public, kube-system이 있으며 namespace를 따로 명시하지 않으면 기본적으로 namespace는 default가 된다. namespace를 생성하는 방법은 다음과 같다. 아래 코드는 양말 쇼핑몰과 관련된 오브젝트들을 묶기위한 namespace이다.
```yaml
#sock-shop-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sock-shop
```

여기서 추가시키는 namespace는 sock-shop과 monitoring이다.

```bash
cd $HOME/kubernetes-tutorial/manifests
kubectl apply -f sock-shop-ns.yaml
kubectl apply -f ../manifests-monitoring/monitoring-ns.yaml
```

---
  ### Pod와 Deployment

  #### Pod
파드는 쿠버네티스의 기본 구성요소이며 쿠버네티스에서 배포할 수 있는 가장 작은 단위이다.  파드와 컨테이너의 다른점은 파드는 단순히 컨테이너 뿐만 아니라, 저장 리소스, 특정 네트워크 설정이나 환경변수, 옵션들을 한꺼번에 모아 캡슐화를 시킬 수 있는 오브젝트이다.

```
#busy-box.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

위의 파일은 쿠버네티스 홈페이지에 올라와 있는 파드의 예시인 busybox.yaml파일이다. 먼저 apiVersion, kind, metadata는 쿠버네티스 오브젝트를 정의하는데 있어 필수적인 요소다. apiVersion를 통하여 kubernetes의 어떤 version의 api를 사용하는지를 명시할  수 있다. apiVersion을 여러개로 나눠둔 이유는 기존의 것을 deprecated 시키기 보다는 새로운 버전을 만들어지면 기존 것도 유지하여 사용할 수 있게 확장성 있게 개발하기 위함이다 . kind는 어떤 오브젝트를 만들지를 명시하는 field이고 metadata는 name, namespace, labels, annotations와 같은 그 오브젝트의 이름, 소속, 분류와 관련된 정보를 명시하는 field다. 그 다음 spec field 아래에서 컨테이너를 정의하게 된다.  spec field의 상세정보는 나중에 설명한다.



#### Deployment
어떤 애플리케이션이 컨테이너로 올라갈지에 대한 것을 정하는 것이 파드였다면 디플로이먼트는 이러한 파드들을 몇 개 띄울 것인지, 어떤 식으로 업데이트할 것인지를 정의할 수 있는 오브젝트다. 주목해야할 점은 컨테이너의 정보를 정의하기 위해 파드를 설정하고 몇 개를 띄울지를 정하기 위해 디플로이먼트를 따로 만드는 것이 아닌 디플로이먼트 안에서 포드 설정, 업데이트 설정까지 다같이 설정을 할 수 있다는 점이다. 그러므로 디플로이먼트는 컨테이너를 정의할 뿐만 아니라 쿠버네티스를 이용해서 어떻게 관리하는 지까지 정의하기 때문에 가장 빈번히 사용되는 오브젝트 중 하나라고 볼 수 있다.

아래는 양말 쇼핑몰 서비스 중 하나인 front-end에 관한 디플로이먼트이다.  spec field 아래에 replicas는 몇 개의 파드를 띄울지, strategy는 이전 파드를 새로운 파드로 교체하는 방식을 기술하는 field입니다. 그 아래 template field에는 위에서 봤던 파드 오프젝트를 정의하는 부분이다. 그래서 디플로이먼트에서 만든 파드들은 이플로이먼트 오브젝트를 이용해서 일괄적으로 관리할 수 있다. 다음은 양말 쇼핑몰에 있는 디플로이먼트 중 하나인 front-end 디플로이먼트이다.

```
#front-end-dep.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: front-end
  namespace: sock-shop
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: front-end
    spec:
      containers:
      - name: front-end
        image: weaveworksdemos/front-end:0.3.12
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
         #limits:
        ports:
        - containerPort: 8079
        env:
        - name: SESSION_REDIS
          value: "true"
        - name: ZIPKIN
          value: zipkin.jaeger.svc.cluster.local
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          capabilities:
            drop:
              - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /
            port: 8079
          initialDelaySeconds: 300
          periodSeconds: 3
        readinessProbe:
          httpGet:
            path: /
            port: 8079
          initialDelaySeconds: 30
          periodSeconds: 3
      nodeSelector:
        beta.kubernetes.io/os: linux
```

이제 양말 쇼핑몰의 디플로이먼트들을 전부 올린다.

```
cd $HOME/kubernetes-tutorial/manifests/deployment
kubectl apply -f ./ 
```



---



  ### Service
#### Service 오브젝트가 필요한 이유
* **파드는 쉽게 사라지고 생겨난다.** 파드들은 각자의 ip주소를 가지고 있다. 이를 통해 서로간의 통신이 가능하다. 하지만 파드들은 쉽게 변경되고 죽고 살아나고 그때마다 ip주소를 변경된다. 만약 서로 통신이 필요한 프론트 엔드 역할을 하는 파드와 백엔드 역할을 하는 파드가 있다고 가정하자. 이 둘은 계속해서 통신을 할 필요가 있다. 그런데 파드들은 쉽게 죽고 살아나서 ip주소가 바뀌기 때문에 계속 통신을 하려면 서로가 변경이 있는지를 계속 추적해야 할 필요성이 생긴다. 이러한 복잡성을 막기위해 서비스라는 오브젝트를 만들어 계속 추적할 필요를 없게 만들었다.
* **한 애플리케이션의 여러 개의 파드(복제) 중 어떤 것을 선택할 지를 어떻게 선택할 것인가?** 이런 것을 고민할 필요가 없이 서비스를 통해 접근을 하면 알아서 파드로 통신을 할 수 있게 해준다.

#### Service의 정의

서비스는 한 애플리케이션에 대한 포드들의 논리적 집합으로서 포드들과 통신하기 위해 여러 개의 포드 중에 어떤 것을 선택할지, ip주소는 무엇인지를 알지 못해도 해당 애플리케이션에 접근할 수 있게 해주는 오프젝트이다. 쿠버네티스는 서비스내에 파드들이 어떻게 바뀌든지 간에 고정된 endpoint를 제공하여 각기 다른 서비스들끼리 변경사항을 추적할 필요 없이 통신을 가능하게 해준다.

#### Service 타입

쿠버네티스의 대부분의 서비스는 클러스터 내부에서만 통신이 이루어지지만 frontend처럼 외부로 노출시킬 필요가 있는 서비스가 있을 수 있다. 그래서 쿠버네티스는 이런 여러가지 상황에서 서비스를 사용하기 위해 4가지 서비스 타입(ServiceTypes)을 제공한다.
1. ClusterIP: 서비스에 클러스터ip(내부ip)를 할당한다. 이 방식은 오직 클러스터 내에서만 서비스가 접근될 수 있다.
2. NodePort:  클러스터ip(내부ip)로만의 접근 뿐만 아니라, NAT가 이용되는 클러스터 내에서 각각 선택된 노드들의 동일한 포트에 서비스를 노출시켜 준다. $NodeIP:$NodePort를 이용하여 클러스터 외부로부터 서비스가 접근할 수 있다. 즉 모든 노드의 30036포트로 서비스를 접근할 수 있다.

  ![nodeport](https://user-images.githubusercontent.com/29041283/58790912-3fe27700-862c-11e9-89e6-cf3ca299442e.PNG)  
  
출처: https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0


3.  LoadBalancer: 클라우드 서비스 제공자의 로드밸런서를 사용하여 서비스에 고정된 공인IP를 할당해준다.

  ![loadbalancer](https://user-images.githubusercontent.com/29041283/58790917-4244d100-862c-11e9-931c-aeb7efac1b70.PNG)  
  
출처: https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0


  4. ExternalName: 서비스를 externalName에 매핑하여 접근할 수 있도록 한다. 이 레포지토리에서는 쓰지 않기 때문에 자세한 건 [홈페이지](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)를 참조한다.
  
#### 사용방법

```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-end
  labels:
    name: front-end
  namespace: sock-shop
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8079
    nodePort: 31125
    protocol: TCP
    name: http
  selector:
    name: front-end
```

서비스는 포드들을 선택하기 위해서 targetport번호와 selector를 이용한다. 위에 사례를 보면 위의 디플로이먼트를 연결하기 위한 front-end 서비스이다. front-end서비스 오브젝트는 포트가 8079이고 name=frontend의 라벨을 가진 포드들을 선택해서 하나의 서비스로 만든다.



이제 양말 쇼핑몰의 service들을 올리고 서비스에 접속해보자.

```bash
cd $HOME/kubernetes-tutorial/manifests/service
kubectl apply -f ./
kubectl get svc -n sock-shop
```

![getservice](https://user-images.githubusercontent.com/29041283/58790928-496bdf00-862c-11e9-8666-aaec14f17eac.PNG)

현재 front-end의 nodeport는 31125이므로 클러스터에 있는 노드의 외부ip중 하나의 ip가 125.209.222.141라고하면 외부에서 접근하기 위해서는 **125.209.222.141:31125** 로 접근하면 된다. 접근을 하면 다음과 같이 양말 쇼핑몰 홈이 나오게 된다.

![sockshop](https://user-images.githubusercontent.com/29041283/58790947-5092ed00-862c-11e9-9928-61dc39b4ae71.PNG)

---

이제 양말 쇼핑몰 서비스와 쿠버네티스를 모니터링하는 Prometheus와 Grafana 서비스를 쿠버네티스에 올리는 과정에서 volume, configmap, secret, 권한설정(RBAC), horizontal pod autoscaler(hpa)에 대해 알아보자.

  ### Volume

#### 볼륨이 필요한 이유
* 컨테이너가 충돌이 일어나 죽으면 그 안에 있는 파일들은 다 사라지게 된다. 이를 안전하게 저장할 방법이 필요하다.
* 또한 파드 안에 컨테이너들끼리 파일을 공유할 필요가 있다.
#### 볼륨의 장점
* 컨테이너가 죽어도 데이터를 저장할 수 있다.
* 다양한 회사가 제공하는 볼륨을 쿠버네티스에 연결할 수 있다.
* 다양한 오브젝트에 들어있는 데이터를 볼륨형태로 쉽게 컨테이너에 마운트 시킬 수 있다. (configmap, secret 등등)
* 여러 개의 볼륨을 한 컨테이너에 쉽게 연결할 수 있다.
#### 볼륨 사용방법

![volume1](https://user-images.githubusercontent.com/29041283/58790996-643e5380-862c-11e9-9a90-01b638fe060d.PNG)
![volume2](https://user-images.githubusercontent.com/29041283/58790998-66081700-862c-11e9-9324-779f44aa0249.PNG)  

디플로이먼트 안에 volumes field에서 어떤 볼륨을 붙일지를 지정을 해주고 volumeMounts에서 해당 볼륨을 마운트 시켜준다.

#### 자주 사용했던 volume 종류
1. emptyDir: 포드가 사라지면 emptyDir도 사라짐. 컨테이너끼리 데이터를 공유하는 용도
2. configMap: configMap 오브젝트에 들어있는 설정파일을 컨테이너 안에 마운트하기 위한 볼륨
3. secret: secret 오프젝트에 있는 데이터를 마운트하기 위한 볼륨
4. persistentVolumeClaim: 컨테이너가 사라져도 데이터가 사라지지 않는 볼륨.


#### Dynamic provisioning과 이를 위해 필요한 오프젝트
1. Dynamic provistioning: volume을 미리 생성해 놓지 않아도 필요하면 동적으로 생성하는 기능
2. StorageClass: 동적으로 어느 곳(aws, azure, local)에서 볼륨을 생성할지를 지정해주는 오브젝트
3. PersistentVolumeClaim(pvc): pvc안에서 StorageClass를 지정하여 어디서 볼륨을 받아올지를 지정하고 얼만큼 받아올지를 정할 수 있는 오브젝트
4. 이 pvc를 deployment에서 붙여주고 사용이 시작되면 자동으로 storage provider에서 volume을 생성해준다.
5. 하지만 로컬에서는 동적으로 생성을 해주지 않으므로 PersistentVolume을 만들어주어야 한다.

#### 로컬에서 volume사용방법(PersistentVolumeClaim) 

1. Storageclass를 생성한다. 여기서 provisioner는 local이므로 kubernetes.io/no-provisioner

```yaml
#/manifests-pv/sc-prometheus.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: prometheus-local-storage
  namespace: monitoring
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

prometheus와 grafana의 storagle class를 apply한다.

```bash
cd $HOME/kubernetes-tutorial/manifests-pv
kubectl apply -f sc-promtheus.yaml
kubectl apply -f sc-grafana.yaml
```

2. 원래 storageclass에서는 provisoner가 다른 cloud provider 회사로 되어 있으면 동적으로 볼륨이 할당되지만 local에서는 따로 persistentvolume를 만들어 주어야 한다. 그래서 local의 어느 경로에 얼마만큼의 용량을 할당할지, 그리고 이를 사용할 storageclass는 무엇인지를 명시해준다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-local-pv
  namespace: monitoring
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: prometheus-local-storage
  local:
    path: /mnt/disk/prometheus-vol
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node1
```

prometheus와 grafana의 pv를 위한 디렉토리를 로컬에 만들고 persistentvolume을 아래와 같이 설정해준다.

```bash
DIRNAME="prometheus-vol"
sudo mkdir -p /mnt/disk/$DIRNAME 
sudo chcon -Rt svirt_sandbox_file_t /mnt/disk/$DIRNAME
sudo chmod 777 /mnt/disk/$DIRNAME
kubectl apply -f pv-promtheus.yaml

DIRNAME="grafana-vol"
sudo mkdir -p /mnt/disk/$DIRNAME 
sudo chcon -Rt svirt_sandbox_file_t /mnt/disk/$DIRNAME
sudo chmod 777 /mnt/disk/$DIRNAME
kubectl apply -f pv-grafana.yaml
```

2. Persistent volume claim을 생성한다. PVC에서는 얼마만큼의 용량이 필요하고 어떤 Storageclass를 사용할지를 명시해준다.    

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: prometheus-claim
  namespace: monitoring
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: prometheus-local-storage
  resources:
    requests:
      storage: 20Gi
```

pvc를 apply해준다.

```bash
kubectl apply -f pvc-promtheus.yaml
kubectl apply -f pvc-grafana.yaml
```

---



  ### Configmap

#### Configmap의 정의
configmap은 여러 어플리케이션에서 필요한 설정파일을 쿠버네티스 상에서 관리하고 해당 어플리케이션에 적용할 수 있게 해주는 오브젝트이다. 
#### 사용방법
1. 어플리케이션에서 필요한 설정 내용을 configmap에서 data에 prometheus의 설정파일을 정의를 해준다.

![cfgmap0](https://user-images.githubusercontent.com/29041283/58791012-6acccb00-862c-11e9-95a1-4571f59c17c4.PNG)

2. 디플로이먼트에서 volumes부분에서 마운트를 시켜주고 volumeMounts에서 해당 설정파일을 컨테이너 안 어느 path에 마운트 시킬지를 결정해준다.

![cfgmap1](https://user-images.githubusercontent.com/29041283/58791023-6ef8e880-862c-11e9-998b-98871492342f.PNG)
![cfgmap2](https://user-images.githubusercontent.com/29041283/58791030-715b4280-862c-11e9-89b1-d0cdd8bf2baf.PNG)

3. 그리고 해당 이미지에 대한 arguments에 config파일이 컨테이너 내에 어디에 있는지를 명시해준다.

![configmap arg](https://user-images.githubusercontent.com/29041283/58791033-74eec980-862c-11e9-84bf-6ad88ae515ea.PNG)

monitoring의 관한 configmap을 배포한다.

```bash
cd $HOME/kubernetes-tutorial/manifests-monitoring
kubectl apply -f prometheus-alertrules.yaml -f prometheus-config4.yaml -f grafana-configmap.yaml -f grafana-alert-configmap.yaml
```



알림 담당을 하는 alertmanager에 대한 configmap설정에서 slack의 api url를 수정해주어야 한다. 아래의 링크를 참조하여 api url를 생성해보자. 여기에서는예시를 'https://hooks.slack.com/services/xxxxxx' 라고 한다.

참조: <https://api.slack.com/incoming-webhooks>

alertmanager에 대한 configmap은 다음과 같다.


```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: alertmanager
  namespace: monitoring
data:
  config.yml: |-
    global:
      slack_api_url: SLACK_URL
    route:
      group_by: [Alertname]
      receiver: slack-notifications
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#monitor'
```


이제 configmap안에 SLACK_URL값에 실제 url을 넣어보자.


```bash
#실제 url를 alertmanager-confimap.yaml안에 있는 SLACK_URL string과 바꿔주는 명령어
SLACK='https://hooks.slack.com/services/xxxxxx'
sed -i -e s,SLACK_URL,"$SLACK",g $HOME/kubernetes-tutorial/manifests-alerting/alertmanager-configmap.yaml

#넣은 후 alertmanger를 배포합니다.
kubectl apply -f alertmanager-configmap.yaml -f alertmanager-dep.yaml -f alertmanager-svc.yaml
```



---




  ### 권한설정-RBAC

#### 쿠버네티스 인증: 
사용자는 kubectl 명령어를 이용해서 쿠버네티스 상의 리소스에 접근할 수 있다. 또 쿠버네티스 상에 올라간 여러 서비스나 포드들도 쿠버네티스의 리소스에 접근할 필요가 있다. 여러 유저와 서비스가 존재하므로 이것들에 대한 접근권한을 지정할 필요가 있다. 이를 지정해고 요청이 들어오면 apiserver에서 이를 확인하여 접근을 가능하게 해준다.

![auth](https://user-images.githubusercontent.com/29041283/58791046-78825080-862c-11e9-87d8-9354c2cb0491.PNG)

#### RBAC(Role-based access control)
쿠버네티스는 일반적으로 RBAC을 사용해서 권한을 부여한다. RBAC이란 룰 기반으로 리소스에 대한 접근 권한을 제어하는 것을 말한다.   
RBAC에서 리소스에 대한 권한을 부여하는 방식은 다음과 같다.

![rbac](https://user-images.githubusercontent.com/29041283/58791058-7b7d4100-862c-11e9-85b9-96aa751f39e8.PNG)

크게 Role, RoleBinding, Subject가 있다고 생각하면 된다. 먼저 Role에서는 여러 자원(resources)과 그것들을 제어할 수 있는 행동들(verbs)을 규정해 놓은 Rules들이 정의되어 있고 Subject는 저 행동들을 사용할 대상이다. 그리고 마지막으로 정의했던 Role과 Subject를 연결시켜주는 것이 RoleBinding이다. 그래서 실질적으로 우리가 만들어야 할 설정파일은 Role, Subject, RoleBinding이다.

##### Role(ClusterRole)
Role은 Role과 ClusterRole이 있는데  Role은 하나의 네임스페이스 안에 있는 리소스에 대한 접근 권한을 정의할 수 있고 ClusterRole은 클러스터 범위의 리소스에 대한 접근 권한을 정의할 수 있다.
```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
  labels:
    app: prometheus
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
- nonResourceURLs:
  - /metrics
  verbs:
  - get
```
위의 ClusterRole은 prometheus 서비스에게 권한을 부여하기 위해 만든 ClusterRole로서 위에 resources 서술된 리소스에 대하여 verbs안에 있는 권한을 가질 수 있는 role이다.

자세한 설정은 [여기](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#review-your-request-attributes)를 참조하자.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
```

그리고 prometheus 서비스가 사용할 service account를 만들어 주고

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
  labels:
    app: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

해당 serviceaccount와 clusterrole을 ClusterRoleBinding 오브젝트에서 연결을 시켜준다.
하나의 clusterrole을 만들어 놓고 여러 user나 serviceaccount에게 binding을 통하여 연결시켜줄 수 있다.

이제 권한 설정을 해보자.

```bash
cd $HOME/manifest-monitoring/
kubectl apply -f prometheus-cr.yml -f prometheus-sa.yml -f prometheus-crb.yml
```

node들의 정보를 수집하는 node-exporter도 배포하자.

```bash
cd node-exporter
kubectl apply -f ./
```

---



  ### 오토스케일링

  #### HPA(Horizontal Pod Autoscaler)
HPA(Horizontal Pod Autoscaler)는 cpu, 메모리 사용량과 기타 요소들을 관찰하여 필요시 자동으로 파드들의 숫자를 조정해주는 오브젝트이다.

  ##### 사용방법
아래 파일은 front-end 디플로이먼트에 대한 오토스케일링을 담당하는 HorizontalPodAutoscaler이다.  spec.scaleTargetRef 안에 해당 Deployment를 넣어주고, Relication을 어떻게 할지 min,max를 넣어준다. metrics은 어떤 리소스가 특정 수치 이상일때 오토스케일이 일어나는 조건을 서술하는 field로 cpu, memory 사용량이 기본이지만 custome한 조건도 추가할 수 있다. 이 hpa는 front-end의 파드가 cpu사용량이 75%이상이 되거나 memory사용량이 95%이상이 되면 자동으로 scale up이 일어나게 된다. 

```yaml
#kubernetes-tutorial/autoscaling/front-end-hsc.yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: front-end
  namespace: sock-shop
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: front-end

  minReplicas: 1
  maxReplicas: 1

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 75

  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 95
```



이제 양말쇼핑몰에 대한 hpa 오브젝트를 apply해보자.

```bash
cd autoscaling
kubectl apply -f ./

kubectl get hpa -n sock-shop
```
`kubectl get hpa -n sock-shop`을 하면 현재 pod들이 사용하는 평균 리소스 사용량과 replication을 확인할 수 있다.

  ![hpa-result](https://user-images.githubusercontent.com/29041283/58791067-80da8b80-862c-11e9-9d76-87b924aec7ab.PNG)
  
  ---



  ### Secret

Secret 오브젝트는 패스워드, 토큰이나 키값 등의 적은 양의 민감한 데이터를 다루는 오브젝트이다. 기존에는 사용하였으나 더이상 사용하지 않기에 설명만 하도록 한다.

#### 만드는 방법
##### 1. 이미 있는 파일 자체를 Secret으로 만들기
```bash
echo -n 'admin' > ./username.txt
echo -n '1f2d1e2e67df' > ./password.txt
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
kubectl get secrets
```
##### 2. 데이터를 암호화시키고 Secret 오브젝트 안에 직접 주입
```bash
$ echo -n 'admin' | base64
YWRtaW4=
$ echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm

$ echo "
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm " > ./secret.yaml
  
$ kubectl apply -f secret.yaml
```

##### 3. string을 통째로 넣기: stringData안에 파일 자체를 전부 넣는 방식

앞서 설명한 alertmanager-configmap을 secret으로 바꿨다.

```yaml
#kubernetes-tutorial/manifests-alerting/alertmanager-secret.yaml
kind: Secret
apiVersion: v1
metadata:
  name: alertmanager
  namespace: monitoring
stringData:
  config.yml: |-
    global:
      slack_api_url: SLACK_URL
    route:
      group_by: [Alertname]
      receiver: slack-notifications
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#monitor'
```



#### 사용방법

deployment안에 볼륨(volumes)을 이용해 파일자체를 마운트 시키거나 생성시 환경변수(env)로 만들어서 사용한다.

![secret2](https://user-images.githubusercontent.com/29041283/58791106-9485f200-862c-11e9-9a8d-dfd0916161b8.PNG)

---

  #### Defendencies
  * os: centos 7.5
  * docker version: 18.09.2
  * go version: 1.10.6
  * kubernetes version: 1.13.3
  * kubespray: 2.8.3
  * istio: 1.1.4
  * helm: 2.13.0
  * metal-lb: 0.7.3
