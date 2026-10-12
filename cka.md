
# k8s - Scheduling

### Scheduling

https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/

* Which node to assign?
* **How scheduling works**
  * What to Schedule?
  * Which node to schedule?
  * (Schedule)Bind Pod to Node -> Create bind object
* **Labels & Selectors**
  * `kubectl get po --selector app=App1`
  * 리소스간 연결을 위해서 사용
  * annotation의 경우 label과 달리 좀 더 information을 담는 용도로 사용
* **Taints & Tolerations**
  * Taints and tolerations are used to set restrictions on what pods can be scheduled on a node.
    * Remember, taints and tolerations does not tell the pod to go to a particular node. --> 이건 node affinity
    * Instead, it tells the node to only accept pods with certain tolerations.
  * taints는 node에 설정 / toleration은 pod에 설정 
    * -> node에 설정된 taint 조건에 대한 toleration 설정을 가진 pod는 해당 노드에 스케줄링 될 수 있음
  * `kubectl taint nodes node-name app=blue:taint-effect`
    * taint-effect
      * NoSchedule
      * PreferNoSchedule
      * NoExecute
  * Toleration 설정
    ```yaml
      tolerations:
      - key: "app"
        operator: "Equal"
        value: "blue"
        effect: "NoSchedule"
    ```
  * 기본적으로는 master node에 taints가 설정되어있음
* **Node Selectors**
  * `kubectl label node node01 size=large`
  * nodel에 label을 추가
  * pod에서 node selector에 명시한 label 해당하는 노드로 할당됨
  * Limitation - selector의 조건이 더 복잡한 경우에는 활용하기 어려움 (ex. size=large or size=medium) --> node affinity 활용
  ```yaml
    nodeSelector:
      size: large
  ```
* **Node Affinity**
  * node selector보다 더 advanced 설정이 가능
  * 하지만 그러다보니 node selector보다 설정이 더 복잡함
  ```yaml
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: size
              operator: In
              values:
              - large
              - medium
            - key: size
              operator: NotIn
              values:
              - small
  ```
  * pod가 node에 스케줄링이 된 상태에서 node의 label이 바뀐다면?
    * nodeAffinity 아래의 property 가 해당 case를 설명
  * Node Affinity Type
    * requiredDuringSchedulingIgnoredDuringExecution
    * preferredDuringSchedulingIgnoredDuringExecution
    * requiredDuringSchedulingRequiredDuringExecution
      * DuringScheduling - 처음 스케줄링될 때 "반드시" 또는 "가능한" affinity에 맞게 스케줄링할건지
      * DuringExecution - 이미 스케줄링되어 동작 중일 때 node label이 수정되면 "무시"할건지 "반드시" affinity를 맞출건지 (evict)
* **Taints/Tolerations vs Node Affinity**
  * 특정 node에 특정 pod만 스케줄링하도록 하고싶을 때
    1. node에 taint를 설정하고 pod에 toleration을 설정
       * node에는 toleration이 설정된 pod만 배치될 수 있다
       * 하지만 그 pod는 taints가 설정되지 않은 다른 node에도 배치될 수 있다?!
    2. node에 label을 설정하고 pod에 node affinity를 설정
       * pod는 그 node에만 배치될 수 있다
       * 그 node에는 다른 pod도 배치될 수 있다?!
  * 따라서 이러한 스케줄링을 위해서는 taints/toleration과 node affinity를 둘 다 설정해줘야 한다.



* **Resource requests and limits**
  * resource request
    * container의 minimum resource
    * 스케줄러는 해당 값을 보고 pod를 node에 배치
  * resource limit
    * container는 default로 resource limit이 안걸려있음
  > 1 CPU는 vCPU를 의미 -> 0.1은 100m을 의미
  > 메모리는 G,K,M Gi,Mi,Ki 모두 사용 가능
  * 만약 pod가 limit을 넘기게 된다면?
    * cpu는 시스템에서 값을 못넘기게 제한시킬 수 있음
    * memory는 시스템에서 제한이 안걸림
      * 따라서 설정된 limit보다 더 많은 메모리를 지속적으로 사용했을 때 OOM 에러와 함께 pod를 죽임 (terminated)
  * Default Configuration
    * Kubernetes does not have a CPU or memory request or limit set
  * LimitRange
    * namespace level object
    * pod내 container들에 default request/limit을 설정해줄 수 있음
    ```yaml
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: cpu-resource-constraint
    spec:
      limits:
      - default: # default limits
          cpu: 500m
        defaultRequest: # default requests
          cpu: 500m
        max: # max and min define the limit range
          cpu: "1"
        min:
          cpu: 100m
        type: Container
    ```
  * Resource Quota
    * namespace level object
    * namespace 내의 container 들에 max/min request/limit을 설정해줄 수 있음
    ```yaml
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: compute-resources
    spec:
      hard:
        requests.cpu: "1"
        requests.memory: "1Gi"
        limits.cpu: "2"
        limits.memory: "2Gi"
        requests.nvidia.com/gpu: 4
    ```
* DaemonSets
  * 노드당 pod가 하나씩 돌도록 만듦





## Static Pods

* kubelet은 기본적으로 kube-apiserver로부터 pod 생성을 요청받고 pod를 생성함
* 하지만 kubelet 그 자체적으로도 pod는 생성이 가능! --> 이것이 static pod!
  * pod가 아닌 replica set, deployment 같은 k8s object는 안됨
* 어떻게 생성?
  * node 내에 설정한 특정 directory에 pod manifest 파일을 넣으면 kubelet이 이걸보고 자동으로 pod 관리함
  * 해당 directory 경로는 kubelet.service 파일에서 `pod-manifest-path` 옵션을 보거나 `config` 옵션에 설정된 config 파일에서 `staticPodPath` 설정을 보면 됨
    * `/var/lib/kubelet/config.yaml`
* kubelet에서 static pod가 생성되면 이는 kube-apiserver에 read-only로 mirror object가 생성돼서 kubectl로 볼 수 있음
  * mirror object 이름에는 node 이름이 postfix로 붙음 (ex. static-web-node01)
* 언제 씀?
  * master 노드에 kubelet을 설치할 때, apiserver, controller, ETCD 등의 control plane component들을 static pod를 이용해서 설치
  * That's how the kube admin tool sets up a Kubernetes cluster



## Multiple Schedulers

* 필요에 맞게 custom scheduler를 만들  수 있음
* default scheduler와 함께 additional scheduler를 k8s cluster에 배포할 수 있음
* pod 생성 시 schedulerName을 지정하여 pod를 스케줄링
* scheduler 생성 방법
  * scheduler를 pod로 배포할 때
    * kube-scheduler 바이너리 (혹은 custom scheduler) 를 실행하는 pod
    * 실행 명령에서 config file option을 지정
    ```yaml
      containers:
      - command:
        - kube-scheduler
        - --address=127.0.0.1
        - --kubeconfig=/etc/kubernetes/scheduler.conf             # it has authentication information to connect to the Kubernetes API server.
        - --config=/etc/kubernetes/my-scheduler-config.yaml
        image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
        name: kube-scheduler
    ```
  * kube-scheduler config file
    * scheduler name 지정
    * leader elect option
      * when you have multiple copies of the scheduler running on different master nodes as a high-availability setup
      * only one can be active at a time
    ```yaml
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    profiles:
    - schedulerName: my-scheduler
    leaderElection:
      leaderElect: true
      resourceNamespace: kube-system
      resourceName: lock-object-my-scheduler
    ```
  * service account / cluster role binding 가 authentication을 위해 필요함
  * config file은 ConfigMap을 통해 volume mount
* Scheduler 로그 확인
  * scheduler pod describe에서 나오는 이벤트 확인
  * kubernetes 이벤트 확인 --> kubectl get events -o wide
  * scheduler pod의 log 확인 --> kubectl log my-scheduler



## Scheduler Profile



* scheduling --> filtering --> scoring --> binding
  * scheduling queue
    * pod가 생성되면 scheduling queue에 들어감
    * priority 기반으로 pod들이 sorting --> priorityClass 리소스 설정을 통해...
  * filter phase
    * pod를 돌릴 수 없는 노드들을 필터링함
    * 리소스 없거나, nodeName, taint+toleration
  * scoring phase
    * pod가 할당될 수 있는 노드들의 점수를 매김
    * 남은 리소스 양, image locality 등등..
  * binding phase
    * pod는 점수가 가장 높은 node에 할당됨
* 위의 과정은 plugin에 의해 수행됨 (한 plugin에 여러 phase에서 동작할 수 있음)
* 아래 그림에 plugin은 파란색, extension point는 보라색으로 표시됨
  * 각 extension point에 plugin을 적절히 배치
  * plugin을 생성해서 custom 스케줄러를 구성할 수 있음

![](X982Iizjbh0e6gxwZ6am8ZcoXr5HgTsjbgHQoRcJrvs=.png)

* Multi scheduler 관련
  * multi scheduler에서 race condition이 생길 수 있음
    * ex. 각 scheduler가 동시에 노드에 남은 리소스를 계산
  * 하나의 scheduler에 multiple profile을 등록할 수 있음
  ![](9rvmb1VZBGul-w7iS4ub2dmfcWvdSEIiu07HgLs_aWA=.png)

---


# k8s- Lifecycle

* ~~Rolling Update / Rollbacks~~
* ~~Configure Application~~
  * Commands and Arguments
  * Environment Variables
* Multi Container Pod
* Autoscaling (HPA / VPA)



## Rolling Update & Rollback

* Rollout status command
  ```
  > kubectl rollout status deployment/ myapp deployment
  > kubectl rollout history deployment/ myapp deployment
  ```
* Deployment Strategy
  * Recreate
  * RollingUpdate
* Rollback command
  * `kubectl rollout undo deployment/ myapp deployment`



## Configure Application

Configuring applications comprises of understanding the following concepts:

* Configuring Command and Arguments on applications
* Configuring Environment Variables
* Configuring Secrets



#### Commands and Arguments

컨테이너가 실행될 때 command를 넘기는 법에 대해서

* Dockerfile
  * Dockerfile의 CMD 필드에 컨테이너 실행 시 수행되는 기본 명령어를 입력함
    ```docker
    FROM Ubuntu
    CMD sleep 5
    # CMD ["sleep", "5"] (O)
    # CMD ["sleep 5"]    (X) --> JSON 형태로 보낼 땐 명령어를 array 형태로 보내야 함
    ```
  * 위의 이미지를 실행할 때 다른 명령어로 overwrite 할 수 있음
    ```
    docker run ubuntu-sleeper sleep 10
    ```
  * 만약 sleep command는 고정으로 쓸거고 argument만 바꾸고싶다면
    ```docker
    FROM Ubuntu
    ENTRYPOINT ["sleep"]
    CMD ["5"]
    ```
    ```
    docker run ubuntu-sleeper 10
    docker run --entrypoint sleep-v2 ubuntu-sleeper 10
    ```
* Pod Definition
  * pod definition에서 command를 overwrite 할 때
    * CMD와 args가 대응되고 ENTRYPOINT와 command가 대응되는 것!!!
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: ubuntu-sleeper-pod
    spec:
      containers:
      - name: ubuntu-sleeper
        image: ubuntu-sleeper
        command: ["sleep-v2"]
        args: ["10"]
    ```



#### Environment Variables

* Pod definition에서 env 설정 방법
  * Key-value / ConfigMap / Secrets
  ```yaml
  spec:
    containers:
    - name: web-color
      image: web-color
      env:
        - name: APP_COLOR
          value: pink
        - name: APP_COLOR2
          valueFrom:
            configMapKeyRef:
        - name: APP_COLOR3
          valueFrom:
            secretKeyRef:
  ```
* ConfigMap
  * config map 또한 imperative 또는 declarative 방식으로 생성할 수 있음
    ```
    kubectl create configmap app-config \
                  --from-literal=APP_COLOR=blue \
                  --from-literal=APP_MOD=prod

    kubectl create configmap app-config \
                  --frome-file=app_config.properties
    ```
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: app-config
    data:
      APP_COLOR: blue
      APP_MODE: prod
    ```
  * ConfigMap in Pods
    * ENV / Single ENV / Volume 으로 injection 가능
    ```yaml
        # ENV
        envFrom:
          - configMapRef:
              name: app-config
        # Single ENV
        env:
          - name: APP_COLOR
            valueFrom:
              configMapKeyRef:
                name: app-config
                key: APP_COLOR
        # Volume
        volumes:
        - name: app-config-volume
          configMap:
            name: app-config
    ```
* Secret
  * ConfigMap과 비슷하지만 data를 encoded 형식으로 저장한다는 차이점 (ex. password)
  ```
  kubectl create secret generic app-secret \
               --from-literal=DB_Host=mysql
               --from-literal=DB_User=root
               --from-literal=DB_Password=paswrd

  kubectl create secret generic app-secret \
               --from-file=app_secret.properties
  ```
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: app-secret
  data:                      # base64 encoding 값을 저장
    DB_Host: bXlzcWw=
    DB_User: cm9vdA==
    DB_Password: cGFzd3Jk
  ```
  * Secret in Pod
    * ENV / Single ENV / Volume 으로 injection 가능
    ```yaml
        # ENV
        envFrom:
          - secretRef:
              name: app-secret
        # Single ENV
        env:
          - name: DB_Host
            valueFrom:
              secretKeyRef:
                name: app-secret
                key: DB_Host
        # Volume
        volumes:
        - name: app-secret-volume
          secret:
            name: app-secret
    ```
  * Note on Secrets
    * Secrets are not Encrypted!! Only Encoded!
      * base64로 encoding된거지 decoding은 누구나 가능함 --> git에 넣고 관리하면 안됨
    * Secrets are note encrypted in ETCD
      * 그러나 ETCD에 저장된 데이터를 encryption할 수 있는 옵션은 있음 ([link](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/))
    * 즉, k8s의 secret은 진짜 secret한 놈은 아니기 때문에 Helm Secret이나 HashiCorp Vault 같은 애들을 쓰는게 권장됨



## Multi Container Pod

* 2개 이상의 container가 pair로 동작하도록 + network / storage 공유 등
* 3 common patterns (CKAD course)
  * side-car pattern
  * adapter pattern
  * ambassador pattern
* InitContainers
  * Pod가 생성될 때 initContainer가 먼저 실행이 끝나고 real container가 시작됨
  * InitContainer도 여러 개를 설정할 수 있음 - 이때 container들은 sequential 한 순서로 실행됨
  * If any of the initContainers fail to complete, Kubernetes restarts the Pod repeatedly until the Init Container succeeds.



## AutoScaling

* CKA 실험에선 아래 2가지에 대해서만 다룸
  * Horizontal Pod Autoscaling (HPA)
  * Vertical Pod Autoscaling (VPA)
  ![](assets/xDpf9GR5-FkK060SdPT18ZE5hMg-3gufTy8xHn2xoWk=.png)
* Scaling Workloads
  * Horizontal Scaling: create more pod
    * Manual: `kubectl scale deployment my-app --replica=3 `
    * Automated: Horizontal Pod Autoscaling (HPA)
  * Vertical Scaling: scale resources to a pod
    * Manual: `kubectl edit ...`
    * Automated: Vertical Pod Autoscaling (VPA)



#### HPA

* continuously monitors the metrics
  * Metrics Server가 설치되어 있어야 함
* automatically increase or decrease the number of pods based on CPU memory or custom metrics
  * ex. CPU 사용량이 증가하면 자동으로 pod를 더 생성

![](assets/hkyfULawxYhbrMdJMstHSiujIw3bwPPyndazhXphLII=.png)

* Imperative
  ```
  > kubectl autoscale deployment my-app \
        --cpu-percent=50 --min=1 --max=10

  > kubectl get hpa
  ```
  * HPA가 생성되어 target의 resource 정의를 체크
  * metrics server로부터 target의 resource 사용량을 체크하여 50%가 넘을 시 replica를 추가로 생성하여 scaling up
  * 사용량이 줄어들면 scaling down
* Declarative
  * HorizontalPodAutoscaler kind를 yaml로 정의하여 생성
    ```yaml
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: php-apache-hpa
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: php-apache
      minReplicas: 1
      maxReplicas: 10
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    ```



#### VPA

* In-place resize of pod resources (Manual)
  * `kubectl edit` 으로 resource 수를 수정하면 기존에 있는 pod를 지우고 새로 생성함
  * Kubernetes v1.27에 alpha 버전으로 반영되어있으며, default로 enable 되지는 않음
    * `InPlacePodVerticalScaling=True` 설정을 해줘야 켜짐
    * beta 버전이 되면 default로 enable 된다고 함
  * 암튼 얘는 VPA 처럼 automated 된 방식은 아님!
  * Limitation
    * cpu, memory 리소스만 수정할 수 있음
    * pod의 QoS class는 수정할 수 없음
    * memory limit은 현재 container가 사용하는 것보다 적게 세팅할 수 없음



* continuously monitors the metrics
* automatically increase or decreases the resources assigned to the pods in a deployment
* VPA 동작 원리
  * HPA와 달리 VPA는 k8s에 built-in 되어있지 않음
  * kubectl apply -f vertical-pod-autoscaler.yaml 로 vpa 관련 component 들을 설치
    * VPA Recommender
      * Metric Server를 모니터링 하면서 optimal CPU, memory 값을 제안
    * VPA Updater
      * pod를 모니터링하면서 Recommender로부터 업데이트가 필요하다고 받으면 pod를 evict 시킴
    * VPA Admission Controller
      * pod를 recreate 할 때 Recommender가 제안한 값으로 resource를 업데이트
* Imperative (X) - built-in 기능이 아니라서!
* Declarative
  * updateMode
    * Off: Recommender만 동작하고 실제 kill, recreate는 안함
    * Initial: kill하진 않고 create 될 때만 Admission Controller가 동작
    * Recreate: resource 사용이 range를 넘어갈 때 evict 시키고 recreate
    * Auto: 현재는 Recreate와 동일하게 동작 (향후에 in-place update가 default로 지원되면 죽이지 않고 바꾸려는 듯)
  ```yaml
  apiVersion: autoscaling.k8s.io/v1
  kind: VerticalPodAutoscaler
  metadata:
    name: my-app-vpa
  spec:
    targetRef:
      apiVersion: "apps/v1"
      kind: Deployment
      name: my-app
    updatePolicy:
      updateMode: "Auto"
    resourcePolicy:
      containerPolicies:
      - containerName: "my-app"
        minAllowed:
          cpu: "250m"
        maxAllowed:
          cpu: "2"
       controlledResources: ["cpu"]
  ```


#### HPA vs VPA

* VPA
  * Stateful workloads, CPU/memory-heavy apps (ex. DB, ML workloads)
* HPA
  * Web apps, microservices, stateless services



