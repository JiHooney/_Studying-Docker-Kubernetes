```bash
지금까지 사용해봤던 디플로이먼트의 각 포드는 별도의 데이터를 가지고 있지 않았고(상태가 없는 stateless), 단순히 요청에 대한 응답만을 반환했다.
그렇지만 데이터베이스처럼 포드 내부에서 특정 데이터를 보유해야 하는(상태가 있는 stateful) 애플리케이션의 경우에는 데이터를 어떻게 관리할지 고민해야 한다.
예를 들어 MySQL 디플로이먼트를 통해 포드를 생성했다고 해도 MySQL 포드 내부에 저장된 데이터는 영속적이지 않다. 
왜냐하면 디플로이먼트를 삭제하면 포드도 함께 삭제되고, 그와 동시에 포드의 데이터 또한 함께 삭제되기 때문이다.

이를 해결하기 위해서는 포드의 데이터를 영속적으로 저장하기 위한 방법이 필요하다. 
도커에서는 docker run -v 옵션이나 docker volume 명령어가 있다. 이러한 옵션들은 단일 컨테이너의 디렉터리를 호스트와 공유함으로써 데이터를 보존했다.
> docker volume create myvolume
> docker run -it --name test -v myvolume:/mnt ubuntu:16.04

쿠버테니스에서도 호스트에 위치한 디렉ㅌ너리를 각 포드와 공유함으로써 데이터를 보존하는 것이 가능하다. 그렇지만 여러 개의 서버로 구성된 쿠버네티스와 같은 클러스터 환경에서는
이 방법이 적합하지 않다. 쿠버네티스는 워커 노드 중 하나를 선택해 포드를 할당하는데, 특정 노드에서만 데이터를 보관해 저장하면 포드가 다른 노드로 옮겨갔을 때
해당 데이터를 사용할 수 없게 된다. 
```

9.1그림

```java
이를 해결할 수 있는 일반적인 방법은 **어느 노드에서도 접근해 사용할 수 있는 퍼시스턴트 볼륨(Persistent Volume)을 사용하는 것이다. 
퍼시스턴트 볼륨은 워커 노드들이 네트워크상에서 스토리지를 마운트해 영속적으로 데이터를 저장할 수 있는 볼륨을 의미한다.**
따라서 포드에 장애가 생겨 다른 노드로 옮겨가더라도 해당 노드에서 퍼시스턴트 볼륨에 네트워크로 연결해 데이터를 계속해서 사용할 수 있다.
네트워크로 연결해 사용할 수 있는 퍼시스턴트 볼륨의 대표적인 예로는 NFS, AWS의 EBS, Ceph, GlusterFS 등이 있다.
```

# 9.1 로컬 볼륨: hostPath, emptyDir

```bash
볼륨을 간단히 사용해 볼 수 있는 hostPath와 emptyDir 두 가지 종류의 볼륨을 먼저 알아본다.

**hostPath는 호스트와 볼륨을 공유하기 위해서 사용하고
empthDir은 포드의 컨테이너 간에 볼륨을 공유하기 위해서 사용한다.**

이 두 가지는 자주 사용하는 볼륨은 아니다.
```

## 9.1.1 워커 노드의 로컬 디렉터리를 볼륨으로 사용: hostPath

```bash
포드의 데이터를 보존할 수 있는 가장 간단한 방법은 호스트의 디렉터리를 포드와 공유해 데이터를 저장하는 것이다.

> mkdir chapter9
> cd chapter9
> vi hostpath-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
    - name: my-container
      image: busybox
      args: [ "tail", "-f", "/dev/null" ]
      volumeMounts:
      - name: my-hostpath-volume
        mountPath: /etc/data
  volumes:
    - name: my-hostpath-volume
      hostPath:
        path: /tmp

volumes 항목에 볼륨을 정의한 뒤, 이를 포드를 정의하는 containers 항목에서 참조해 사용한다.
volumes에서 hostPath 항목을 /tmp로 정의하고 containser에서 volumeMounts항목에서 mountPath를 /etc/data와 매칭이 되도록 설정하였다.

포드를 생성한 뒤 포드의 컨테이너 내부로 들어가 /etc/data 디렉터리에 파일을 생성하면 호스트의 /tmp 디렉터리에 파일이 저장된다.

> kubectl apply -f hostpath-pod.yaml
> kubectl exec -it hostpath-pod touch /etc/data/mydata

#포드가 생성된 워커노드에서 /tmp 디렉터리를 확인
> ls /tmp/mydata

그렇지만 이러한 방식의 데이터 보존은 바람직하지 않다. 디플로이먼트의 포드에 장애가 생겨 다른 노드로 포드가 옮겨갔을 경우,
이전 노드에 저장된 데이터를 사용할 수 없기 때문이다.
```

## 9.1.2 포드 내의 컨테이너 간 임시 데이터 공유: emptyDir

```bash
emptyDir 볼륨은 포드의 데이터를 영속적으로 보존하기 위해 외부 볼륨을 사용하는 것이 아닌,
포드가 실행되는 도중에만 필요한 휘발성 데이터를 각 컨테이너가 함께 사용할 수 있도록 임시 저장 공간을 생성한다.
emptyDir은 비어있는 상태로 생성되며 포드가 삭제되면 emptyDir에 저장돼 있던 데이터도 함께 삭제된다.

간단하게 아파치 웹 서버를 실행하는 포드를 생성해 확인해본다.
아래 yaml파일에서 아파치 웹 서버의 루트 디렉터리(htdocs)를 emptyDir에 마운트함과 동시에 
이 디렉터리를 content-creator 컨테이너의 /data 디렉터리와 공유한다.

> vi empty-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: content-creator
    image: alicek106/alpine-wget:latest
    args: ["tail", "-f", "/dev/null"]
    volumeMounts:
    - name: my-emptydir-volume
      mountPath: /data                      # 1. 이 컨테이너가 /data 에 파일을 생성하면

  - name: apache-webserver
    image: httpd:2
    volumeMounts:
    - name: my-emptydir-volume
      mountPath: /usr/local/apache2/htdocs/  # 2. 아파치 웹 서버에서 접근 가능합니다.

  volumes:
    - name: my-emptydir-volume
      emptyDir: {}                             # 포드 내에서 파일을 공유하는 emptyDir

content-creator 컨테이너 내부로 들어가 /data 디렉터리에 웹 콘텐츠를 생성하면 아파치 웹 서버 컨테이너의
htdocs 디렉터리에도 동일하게 웹 콘텐츠 파일이 생성된다. 

> kubectl apply -f emptydir-pod.yaml
> kubectl exec -it emptydir-pod -c content-creator sh
/# echo Hello, kubernetes! >> /data/test.html
/# exit

> kubectl describe pod emptydir-pod | grep IP

> kubectl run -i --tty --rm debug \
--image=alicek106/ubuntu:curl --restart=Never -- curl $IP주소/test.html
```

# 9.2 네트워크 볼륨

그림9.3

```bash
네트워크 볼륨의 위치는 특별히 정해진 곳이 없다. 네트워크로 접근할 수만 있으면 쿠버네티스 클러스터 내부, 외부 어느 곳에 존재해도 크게 상관이 없다.
단 AWS의 EBS와 같은 클라우드에 종속적인 볼륨을 사용하려면 AWS에서 쿠버네티스 클러스터를 생성할 때 특정 클라우드를 위한 옵션이 별도로 설정돼 있어야 한다.

이번 장에서는 간단히 NFS 볼륨의 사용법만 확인한다.
네트워크 볼륨의 선택 기준은 일반적으로 데이터의 읽기 및 쓰기 속도, 마운드방식(1:1, 1:n), 네트워크 볼륨 솔루션 구축 비용 등을 고려할 수 있다.
```

- NFS를 네트워크 볼륨으로 사용하기

```bash
NFS(Network File System)는 대부분의 운영 체제에서 사용할 수 있는 네트워크 스토리지로, 여러 개의 클라이언트가 동시에 마운트해 사용할 수 있다는 특징이 있다.
NFS는 여러 개의 스토리지를 클러스터링하는 다른 솔루션에 비해 안정성이 떨어질 수는 있으나, 하나의 서버만으로 간편하게 사용할 수 있고,
NFS를 마치 로컬 스토리지처럼 사용할 수 있다는 장점이 있다.

NFS를 사용하려면 NFS 서버와 NFS 클라이언트가 각각 필요하다. 
NFS 서버는 영속적인 데이터가 실제로 저장되는 네트워크 스토리지 서버이고, 
NFS 클라이언트는 NFS 서버에 마운트해 스토리지에 파일을 읽고 쓰는 역할을 한다.

NFS 클라이언트는 워커 노드의 기능을 사용할 것이므로 따로 준비가 필요없고, NFS 서버만 구축하면 된다.

쿠버네티스 클러스터 내부에서 임시 NFS 서버를 하나 생성해 본다.

> vi nfs-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
spec:
  selector:
    matchLabels:
      role: nfs-server
  template:
    metadata:
      labels:
        role: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: gcr.io/google_containers/volume-nfs:0.8
        ports:
          - name: nfs
            containerPort: 2049
          - name: mountd
            containerPort: 20048
          - name: rpcbind
            containerPort: 111
        securityContext:
          privileged: true

> vi nfs-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: nfs-service
spec:
  ports:
  - name: nfs
    port: 2049
  - name: mountd
    port: 20048
  - name: rpcbind
    port: 111
  selector:
    role: nfs-server

> kubectl apply -f nfs-deployment.yaml
> kubectl apply -f nfs-service.yaml

NFS 서버를 위한 디플로이먼트와 서비스를 생성했다면 다음은 해당 NFS 서버의 볼륨을 포드에서 마운트해 데이터를 영속적으로 저장하는 것이다.
NFS 서버를 컨테이너에 마운트하는 포드를 생성한다.

> vi nfs-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: nfs-pod
spec:
  containers:
    - name: nfs-mount-container
      image: busybox
      args: [ "tail", "-f", "/dev/null" ]
      volumeMounts:
      - name: nfs-volume
        mountPath: /mnt           # 포드 컨테이너 내부의 /mnt 디렉터리에 마운트합니다.
  volumes:
  - name : nfs-volume
    nfs:                            # NFS 서버의 볼륨을 포드의 컨테이너에 마운트합니다.
      path: /
      server: {NFS_SERVICE_IP}

mountPath를 /mnt로 설정했기 때문에 NFS 서버의 네트워크 볼륨은 포드 컨테이너의 /mnt 디렉터리에 마운트된다. 
**즉, 컨테이너 내부에서 /mnt 디렉터리에 파일을 저장하면 실제로는 NFS 서버에 데이터가 저장된다.** 
또한 volumes 항목에서 nfs라는 항목을 정의함으로써 NFS 서버의 볼륨을 사용한다고 명시했다.

위의 YAML 파일에서 유의해야 할 부분은 server 항목이 nfs-service라는 서비스의 DNS이 아닌 {NFS_SERVICE_IP}로 설정돼 있다는 것이다.
NFS 볼륨의 마운트는 컨테이너 내부가 아닌 워커 노드에서 발생하므로 서비스의 DNS 이름으로 NFS 서버에 접근할 수 없다.
노드에서는 포드의 IP로 통신은 할 수 있지만, 쿠버네티스의 DNS를 사용하도록 설정돼 있지는 않기 때문이다.
따라서 이번에는 예외적으로 NFS 서비스의 ClusterIP를 직접 얻은 뒤, YAML 파일에 사용하는 방식으로 포드를 생성해 본다.

# NFS 서버에 접근하기 위한 서비스의 ClusterIP를 얻는다.
> export NFS_CLUSTER_IP=$(kubectl get svc/nfs-service -o jsonpath='{.spec.clusterIP}')

# nfs-pod의 server항목을 NFS_CLUSTER_IP로 교체해서 생성한다.
> cat nfs-pod.yaml | sed "s/{NFS_SERVICE_IP}/$NFS_CLUSTER_IP/g" | kubectl apply -f -

===* 참고===
kubectl get 명령어의 -o jsonpath는 리소스의 특정 정보만 가져올 수 있는 편리한 옵션이다.
포트부분만 가져오고 싶다면 다음과 같다.
-o jsonpath='{.spec.ports}'
===========

포드가 정상적으로 실행됐는지 확인한 뒤, 포드 내부로 들어가 저장 공간이나 마운트 목록을 출력해보면
NFS 서버로부터 저장 공간을 마운트해 사용하고 있음을 알 수 있다. 

> kubectl get pod nfs-pod

> kubectl exec -it nfs-pod sh
/# df -h

===* 참고===
NFS 서버에 마운트하려면 워커 노드에서 별도의 NFS 패키지를 설치해야 할 수도 있다. 
만약 포드가 ContainerCreating 상태에서 더 이상 진행되지 않는다면 kubectl describe pods 명령어로
무엇이 문제인지 파악하고, 포드가 할당된 워커 노드에서 아래 명령어로 NFS와 관련된 패키지를 설치한다.
> apt-get install nfs-common
===========

NFS 서버가 /mnt 디렉터리에 마운트됐으므로 포드의 컨테이너 내부의 /mnt 디렉터리에 저장된 파일은 포드가 다른 노드로 옮겨가거나
포드를 재시작해도 삭제되지 않는다. 네트워크로 접근할 수 있는 볼륨인 NFS 서버에 파일이 영속적으로 저장되기 때문이다.

실제 운영 환경에서는 NFS 서버를 도입하려면 백업 스토리지를 별도로 구축해 NFS의 데이터 손실에 대비하거나, 
NFS 서버의 설정 튜닝 및 NFS 서버에 접근하기 위한 DNS 이름을 준비해야 한다.
```

# 9.3 PV, PVC를 이용한 볼륨 관리

## 9.3.1 퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임을 사용하는 이유

```bash
쿠버네티스에서 지원하는 대부분의 볼륨 타입은 포드나 디플로이먼트의 YAML 파일에서 직접 정의해 사용할 수 있다.
하지만 실제로 애플리케이션을 개발한 뒤 YAML 파일로 배포할 때는 이러한 방식이 바람직하지 않을 수도 있다.

예를 들어 MySQL 디플로이먼트를 배포하기 위한 YAML 파일을 작성한다고 했을 때 
MySQL은 상태를 가지는 애플리케이션이기 때문에 반드시 영속적 스토리지를 마운트해 데이터를 보관해야 한다.
그래서 NFS 네트워크 볼륨을 사용하기로 해서 이를 YAML 파일에 기입했다고 해보자.

만약 내 부서 내에서만 MySQL 디플로이먼트를 사용할 것이라면 문제 없겠지만,
이 YAML 파일을 다른 개발 부서에 배포하거나 웹상에 공개해야 한다면 문제가 생긴다.
MySQL 디플로이먼트의 YAML 파일에 네트워크 볼륨으로서 NFS를 고정적으로 명시해뒀기 때문에 
해당 YAML 파일로 MySQL 을 생성하려면 반드시 NFS를 사용해야 한다.
게다가 MySQL의 데이터를 보관하기 위해 iSCSI나 GlusterFS를 사용하고 싶다면 해당 네트워크 볼륨 타입을 명시하는 
별도의 YAML 파일을 여러 개 만들어 배포해야 한다.
**즉, 볼륨과 애플리케이션의 정의가 서로 밀접하게 연관돼 있어 서로 분리할 수 없는 상황이 된다.**

이러한 불편함을 없애기 위해 퍼시스턴트 볼륨(Persistent Volume: PV)과 퍼시스턴트 볼륨 클레임(Persistent Volume Claim: PVC)라는 오브젝트를 제공한다.
**이 두 개의 오브젝트는 포드가 볼륨의 세부적인 사항을 몰라도 볼륨을 사용할 수 있도록 추상화해주는 역할을 담당한다.
즉, 포드는 생성하는 YAML 입장에서는 네트워크 볼륨이 NFS인지, AWS의 EBS인지 상관없이 볼륨을 사용할 수 있도록 하는 것이 핵심 아이디어이다.**

```

그림9.5

```java
우선 쿠버네티스 클러스터를 관리하는 인프라 관리자와 애플리케이션을 배포하려는 사용자(개발자)가 나뉘어 있다고 가정한다.
1. 인프라 관리자는 네트워크 볼륨의 정보를 이용해 퍼시스턴트 볼륨 리소스를 미리 생성해둔다.
2. 사용자(개발자)는 포드를 정의하는 yaml 파일에 '이 포드는 데이터를 영속적으로 저장해야 하므로 마운트할 수 있는 외부 볼륨이 필요하다'
라는 의미의 퍼시스턴트 볼륨 클레임을 명시하고, 해당 퍼시스턴트 볼륨 클레임을 생성한다.
3. 쿠버네티스는 기존에 인프라 관리자가 생성해뒀던 퍼시스턴트 볼륨의 속성과 사용자가 요청한 퍼시스턴트 볼륨 클레임의 요구 사항이 일치한다면
두 개의 리소스를 매칭시켜 바인드(bind)한다. 포드가 이 퍼시스턴트 볼륨 클레임을 사용함으로써 포드의 컨테이너 내부에 볼륨이 마운트된 상태로 생성된다.

위 과정에서 중요한 점은 **'사용자는 디플로이먼트의 yaml파일에 볼륨의 상세한 스펙을 정의하지 않아도 된다.'**는 것이다.
사용자는 yaml 파일에서 '이 디플로이먼트는 볼륨을 마운트할 수 있어야 한다.'는 의미의 퍼시스턴트 볼륨 클레임을 명시할 뿐이며, 
실제로 마운트되는 볼륨이 무엇인지 알 필요가 없다.

아래는 yaml 파일의 차이점이다.
```

그림 9.6

## 9.3.2 퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임 사용하기

```bash
# pv와 pvc 리소스 출력하기
> kubectl get pv, pvc

이번 장에서는 AWS에서 제공하는 클라우드 플랫폼의 볼륨을 사용해본다. EBS

지금부터 인프라 관리자의 입장에서 외부 네트워크 볼륨을 쿠버네티스에 등록하고
사용자(개발자)입장에서 퍼시스턴트 볼륨 클레임을 사용해본다.
```

- AWS에서 EBS를 퍼시스턴트 볼륨으로 사용하기

```bash
먼저 EBS를 생성한다. awscli 명령어를 이용한다.

> export VOLUME_ID=$(aws ec2 create-volume --size 5 \
--region ap-northeast-2 \
--availability-zone ap-northeaset-2a \
--volume-type gp2 \
--tag-specifications \
'ResourceType=volume,Tags=[{Key=KubernetesCluster,Value=mycluster.k8s.local}]' \
| jq '.VolumeId' -r)

> echo $VOLUME_ID

EBS의 볼륨 ID를 $VOLUME_ID라는 셸 변수에 저장해 두었다. 이 ID를 통해 퍼시스턴트 볼륨을 생성한다.
EBS의 가용영역과 리전은 반드시 쿠버네티스 워커 노드와 동일한 곳에 있어야 한다.

방금 생성한 EBS 볼륨을 통해 쿠버네티스의 퍼시스턴트 볼륨을 생성해 본다. 

> vi ebs-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv
spec:
  capacity:
    storage: 5Gi         # 이 볼륨의 크기는 5G입니다.
  accessModes:
    - ReadWriteOnce    # 하나의 포드 (또는 인스턴스) 에 의해서만 마운트 될 수 있습니다.
  awsElasticBlockStore:
    fsType: ext4
    volumeID: <VOLUME_ID>

EBS를 생성할 때 셸 변수에 저장했던 VOLUME_ID 값을 이용해 퍼시스턴트 볼륨을 생성해본다.
> cat ebs-pv.yaml | sed "s/<VOLUME_ID>/$VOLUME_ID/g" | kubectl apply -f -

#pv목록 출력
> kubectl get pv

이번에는 사용자 입장에서 퍼시스턴트 볼륨 클레임과 포드를 함께 생성해본다.
아래의 yaml 파일은 my-ebs-pvc라는 퍼시스턴트 볼륨 클레임을 먼저 생성한 뒤, 이를 포드의 volumes 항목에서 사용함으로써
포드 내부에 EBS 볼륨을 마운트한다.

> vi ebs-pod-pvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-ebs-pvc                  # 1. my-ebs-pvc라는 이름의 pvc 를 생성합니다.
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce       # 2.1 속성이 ReadWriteOnce인 퍼시스턴트 볼륨과 연결합니다.
  resources:
    requests:
      storage: 5Gi          # 2.2 볼륨 크기가 최소 5Gi인 퍼시스턴트 볼륨과 연결합니다.
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-mount-container
spec:
  containers:
    - name: ebs-mount-container
      image: busybox
      args: [ "tail", "-f", "/dev/null" ]
      volumeMounts:
      - name: ebs-volume
        mountPath: /mnt
  volumes:
  - name : ebs-volume
    persistentVolumeClaim:
      claimName: my-ebs-pvc    # 3. my-ebs-pvc라는 이름의 pvc를 사용합니다.

accessModes와 resources 항목은 볼륨의 요구사항으로, 해당 조건을 만족하는 퍼시스턴트 볼륨과 연결돼야 한다는 의미이다. 

위 yaml 파일을 이용해 포드와 퍼시스턴트 볼륨 클레임을 생성한다.
> kubectl apply -f ebs-pod-pvc.yaml

> kubectl get pv,pvc

> kubectl get pods

퍼시스턴트 볼륨과 퍼피스턴트 볼륨 클레임의 상태가 bound로 설정됐다면 두 리소스가 성공적으로 연결된 것이다.
포드가 정상적으로 생성되어 Running 상태라면 EBS 볼륨 또한 포드 내부에 정상적으로 마운트됐다는 뜻이다.

> kubectl exec ebs-mount-container -- df -h | grep /mnt

지금까지 사용해 본 퍼시스턴트 볼륨의 사용방법을 정리하면 아래와 같다.
```

그림9.8

```java
1. 포드의 데이터를 영속적으로 저장하기 위해 AWS에서 EBS볼륨을 생성.
2. ebs-pv.yaml 파일을 이용해 1번에서 생성한 EBS 볼륨을 쿠버네티스에서 퍼시스턴트 볼륨으로 등록.
	이때 ebs-pv.yaml 파일에는 awsElasticBlockStore 항목에 EBS의 볼륨ID를 명시. 또한 볼륨의 읽기 및 쓰기 속성, 볼륨크기를 설정.
3. ebs-pod-pvc.yaml파일에서 퍼시스턴트 볼륨 클레임을 먼저 정의해 생성.퍼시스턴트 볼륨 클레임에는 원하는 볼륨의 조건을 나열.
4. 2번에서 생성한 퍼시스턴트 볼륨의 속성이 3번에서 생성한 퍼시스턴트 볼륨 클레임의 요구 조건과 일치하기 때문에
두 리소스가 연결된다.
5. ebs-pod-pvc.yaml 파일에 정의된 포드는 4번에서 생성한 퍼시스턴트 볼륨 클레임을 사용하도록 설정돼 있다.

이러한 방법의 장점을 다시 한 번 정리하면 애플리케이션을 배포하는 입장에서는 볼륨의 세부 구현 및 스펙을 알 필요없이
볼륨 사용에 대한 추상화된 인터페이스를 제공받을 수 있다.
```

## 9.3.3 퍼시스턴트 볼륨을 선택하기 위한 조건 명시

```bash
퍼시스턴트 볼륨 클레임을 사용하면 실제로 볼륨이 어떠한 스펙을 가졌는지 알 필요가 없지만,
사용하려는 볼륨이 애플리케이션에 필요한 최소한의 조건을 맞출 필요는 있을 것이다.

실제 볼륨의 구현 스펙까지는 아니더라도, 적어도 특정 조건을 만족하는 볼륨만을 사용해야 
한다는 것을 쿠버네티스에 알려줄 필요가 있다.

퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임이 accessMode 및 볼륨 크기 속성이 부합해야만
쿠버네티스 두 리소스를 매칭해 바인드합니다.
이전에 사용해봤던 NFS는 여러 개의 인스턴스에 의해 마운트가 가능하지만(1:N 마운트),
AWS의 EBS는 하나의 인스턴스에 의해서만 마운트될 수 있다.(1:1 마운트)
```

그림9.9

- accessModes와 볼륨크기, 스토리지 클래스, 라벨 셀렉터를 이용한 퍼시스턴트 볼륨 선택

```bash
accessModes는 퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임을 생성할 때 설정할 수 있는 속성으로,
볼륨에 대해 읽기 및 쓰기 작업이 가능한지, 여러 개의 인스턴스에 의해 마운트될 수 있는지 등을
의미한다.

**accessModes 이름     kubectl get에서 출력되는 이름     속성설명**
ReadWriteOnce        RWO                              1:1 마운트만 가능, 읽기 쓰기 가능
ReadOnlyMany         ROX                              1:N 마운트 가능, 읽기 전용
ReadWriteMany        RWX                              1:N 마운트 가능, 읽기 쓰기 가능

EBS 볼륨은 기본적으로 읽기, 쓰기가 모두 가능하며 1:1 관계의 마운트만 가능하기 때문에
ReadWriteOnce를 사용했다.
```

예제9.6,6.7 그림

```java
그 외에도 스토리지 클래스나 라벨셀렉터를 이용해 퍼시스턴트 볼륨의 선택을 좀 더 세분화할 수 있다.
스토리지 클래스는 볼륨의 대표 속성 등을 나타내는 것으로,
퍼시스턴트 볼륨을 생성할 때 클래스를 설정하면 해당 클래스를 요청하는 퍼시스턴트 볼륨 클레임과
연결해 바인드한다.
```

예제9.8, 9.9

```java
위의 두 개의 YAML에서는 각각 storageClassName이라는 항목에 my-ebs-volume이라는 값을 입력했다.
이러한 경우 스토리지 클래스의 이름이 일치하는 퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임이 서로 연결된다.
단, storageClassName을 별도로 YAML파일에 명시하지 않으면 클래스가 설정되지 않았다는 뜻인 ""(값이없음)
으로 설정되며, 똑같이 스토리지 클래스가 설정되지 않은 퍼시스턴트 볼륨 또는 퍼시스턴트 볼륨 클레임과 매칭된다.

또는 아래와 같이 라벨 셀렉터를 사용할 수 있다.
이전에 서비스와 디플로이먼트를 서로 연결할 때 라벨 셀렉터를 사용했던 것처럼
퍼시스턴트 볼륨 클레임에 라벨 셀렉터인 matchLabels항목을 정의함으로써
특정 퍼시스턴트 볼륨과 바인드하는 것도 가능하다.
```

예제 9.10, 9.11

## 9.3.4 퍼시스턴트 볼륨의 라이프사이클과 Reclaim Policy

```bash
퍼시스턴트 볼륨과 연결된 퍼피스턴트 볼륨 클레임을 삭제하면 어떻게 되는가?
퍼시스턴트 볼륨과 연결되어 있는 포드와 퍼시스턴트 볼륨 클레임을 삭제했더니
퍼시스턴트 볼륨의 상태가 Available이 아닌 Released라는 상태로 변경된다.
이는 해당 퍼시스턴트 볼륨의 사용이 끝났다는 것을 의미하며,
Released 상태에 있는 퍼시스턴트 볼륨은 다시 사용할 수 없다. 
그렇지만 실제 데이터는 볼륨 안에 고스란히 보존돼 있기 때문에 퍼시스턴트 볼륨을 삭제한 뒤
다시 생성하면 Available상태의 볼륨을 다시 사용할 수 있다.

> kubectl delete -f ebs-pv.yaml

> cat ebs-pv.yaml | sed "s/<VOLUME_ID>/$VOLUME_ID/g" | kubectl apply -f -

> kubectl get pv

하지만 퍼시스턴트 볼륨 클레임을 삭제했을 때, 퍼시스턴트 볼륨의 데이터를 어떻게 처리할 것인지
별도로 정의할 수도 있다. 퍼시스턴트 볼륨의 사용이 끝났을 때 해당 볼륨을 어떻게 초기화할 것인지
별도로 설정할 수 있는데, 쿠버네티스에서는 이를 **Reclaim Policy**라고 부른다.
이 정책에는 크게 Retain, Delete, Recycle 방식이 있다.

쿠버네티스는 기본적으로 퍼시스턴트 볼륨의 데이터를 보존하는 방식인 Retain을 사용한다.
> kubectl get pv 
에서 출력되는 RECLAIM POLICY 항목의 Retain이라는 값이 바로 이것이다.

Retain과 반대의 방식이 Delete, Recycle이 있다.
Delete의 경우 퍼시스턴트 볼륨의 사용이 끝난 뒤에 자동으로 퍼시스턴트 볼륨이 삭제된다.
실습은 아래와 같다.

> vi ebs-pv-delete.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv-delete
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    fsType: ext4
    volumeID: <VOLUME_ID>
  persistentVolumeReclaimPolicy: Delete

#작성한 YAML파일로 퍼시스턴트 볼륨 생성
> cat ebs-pv-delete.yaml | sed "s/<VOLUME_ID>/$VOLUME_ID/g" | kubectl apply -f -

> kubectl get pv

#퍼시스턴트 볼륨 클레임과 포드를 생성
> kubectl apply -f ebs-pod-pvc.yaml

> kubectl get pods

#pv와 pvc가 정상적으로 바인드됐는지 확인
> kubectl get pv,pvc

#연결된 pvc를 삭제함으로써 pv의 사용을 종료
> kubectl delete -f ebs-pod-pvc.yaml

#pv도 함께 삭제됐는지 확인
> kubectl get pv,pvc

Recycle은 pvc가 삭제됐을 때 pv의 데이터를 모두 삭제한 뒤 Available 상태로 만들어준다.
Delete와 마찬가지로 실제로 저장돼 있던 데이터를 모두 삭제한다는 점은 같지만,
퍼시스턴트 볼륨이나 외부 스토리지 자체를 삭제하지는 않는다.
하지만 Recycle 정책은 이제 사용되지 않는다.
```

## 9.3.5 StorageClass와 Dynamic Provisioning

- 다이나믹 프로비저닝과 스토리지 클래스

```bash
지금까지 퍼시스턴트 볼륨을 사용하려면 미리 외부 스토리지를 준비해야 했다.
하지만 매번 이렇게 볼륨 스토리지를 직접 수동으로 생성하고,
스토리지에 대한 접근 정보를 YAML 파일에 적는 것은 귀찮은 일이다.

이를 위해 쿠버네티스는 다이나믹 프로비저닝이라는 기능을 제공한다.
**다이나믹 프로비저닝은 퍼시스턴트 볼륨 클레임이 요구하는 조건과 일치하는 퍼시스턴트 볼륨이
존재하지 않는다면 자동으로 퍼시스턴트 볼륨과 외부 스토리지를 함께 프로비저닝하는 기능이다.**

따라서 다이나믹 프로비저닝을 사용하면 EBS와 같은 외부 스토리지를 직접 미리 생성해 둘 필요가 없다.
```

그림 9.12

```java
1. 위 예시에서 'fast'라는 이름의 스토리지 클래스에는 SSD를 생성하라는 설정을,
 'slow'라는 스토리지 클래스에는 HDD를 생성하라는 설정을 미리 정의했다고 가정한다.
2. 퍼시스턴트 볼륨 클레임에 특정 스토리지 클래스를 명시해 생성한다.
 하지만 pvc의 AccessMode나 Capacity 등의 조건과 일치하는 PV가 존재하지 않는 상태이다.
3. 조건에 일치하는 pv를 새롭게 만들기 위해 스토리지 클래스에 정의된 속성에 따라서 
 외부 스토리지를 생성한다. 만약 pvc에 storageClassName: fast와 같이 설정했고,
 fast라는 이름의 스토리지 클래스가 SSD에 대한 설정 정보를 담고 있다면 SSD 스토리지가 
 동적으로 생성된다.
4. 새롭게 생성된 외부 스토리지는 쿠버네티스의 PV로 등록되고, Pvc와 바인딩된다.

단, 다이나믹 프로비저닝을 모든 쿠버네티스 클러스터에서 범용적으로 사용할 수 있는 것은 아니다.
다이나믹 프로비저닝 기능이 지원되는 스토리지 프로비저너가 미리 활성화돼 있어야 한다.

이번 장에서는 AWS에서 kops로 설치한 쿠버네티스에서 EBS볼륨을 이용해 테스트해 본다.
```
