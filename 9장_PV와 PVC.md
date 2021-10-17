```bash
퍼시스턴트 볼륨: 워커 노드들이 네트워크상에서 스토리지를 마운트해 영속적으로 데이터를 저장할 수 있는 
							볼륨을 의미.

쿠버네티스는 퍼시스턴트 볼륨을 사용하기 위한 기능을 자체적으로 제공한다.
```

# 9.1 로컬 볼륨: hostPath, emptyDir

```bash
HostPath는 호스트와 볼륨을 공유하기 위해서 사용하고

emptyDir은 포드의 컨테이너 간에 볼륨을 공유하기 위해서 사용합니다.
```

## 9.1.1 워커 노드의 로컬 디렉터리를 볼륨으로 사용: hostPath

```bash
Yaml

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

HostPath 항목을 정의함으로써 호스트의 /tmp를 포드의 /etc/data에 마운트했음.

포드 컨테이너의 /etc/data와 호스트의 /tmp는 동일한 디렉토리로써 사용되는 것입니다.

kubectl apply -f hostpath-pod.yaml

kubectl exec -it hostpath-pod touch /etc/data/mydata

ls /tmp/mydata

이러한 방식의 데이터 보존은 바람직하지 않다고 함.

hostPath 볼륨은 모든 노드에 배치해야하는 특수한 포드의 경우에 유용하게 사용할 수 있음.

hosthPath를 사용하는 것은 보안 및 활용성 측면에서 그다지 바람직하지 않으므로 hostPath 를 사용하는 것은 
신중히 고려해 보는 것이 좋다고 함.

```

## 9.1.2 포드 내의 컨테이너 간 임시 데이터 공유: emptyDir

```jsx

emptyhDir : 포드의 데이터를 영속적으로 보존하기 위해 외부 볼륨을 사용하는 것이 아닌, 포드가
						실행되는 도중에만 필요한 휘발성 데이터를 각 컨테이너가 함께 사용할 수 있도록 임시 
						저장 공간 생성

emptyDir 디렉토리는 비어있는 상태로 생성되며 포드가 삭제되면 emptyDir에 저장돼 있던 데이터도 함께
					삭제된다.

아래의 yaml 파일은 아파치 웹 서버의 루트 디렉터리를 emptyDir에 마운트 함과 동시에
content-cretor 컨테이너의 /data 디렉터리와 공유됨.

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

kubectl apply -f emptydir-pod.yaml

kubectl exec -it emptydir-pod -c content-creator sh

kubectl describe pod emptydir-pod | grep IP

kubectl run -i --tty --rm debug \ 
--image=alicek106/ubuntu:curl --restart=Nerver -- curl ip/test.html
```

# 9.2 네트워크 볼륨

```bash

```

- NFS를 네트워크 볼륨으로 사용하기

```bash

```

# 9.3 PV, PVC를 이용한 볼륨 관리

### 9.3.1 퍼시스턴트 볼륨(PV)과 퍼시스턴트 볼륨 클레임(PVC)을 사용하는 이유

🔏 **PV와 PVC 란?** 

포드(여러 개의 컨테이너)가 볼륨의 세부사항을 알지 못해도 볼륨을 사용할 수 있도록 추상화하는 오브젝트 

---

### 9.3.2 퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임 사용하기

```bash
$ export VOLUME_ID=&(aws ec2 create-volume --size 5 \
--region ap-northeast-2 \
--availability-zone ap-northeast-2a \
--volume-type gp2 \
--tag-specifications \
'ResourceType=volume, Tags=[{Key=kubernetesCluster, Value=mycluster.k8s.local}]' \
| jq '.VolumeId' -r)

$ echo $VOLUME_ID
```

AWS 에서 EBS를 퍼시스턴트 볼륨으로 사용하기

```bash
# 리소스 목록 출력 (오브젝트 이름으로 -> 너무 길다)
$ kubectl get persistentvolume,persistentvolumeclaim

# 리소스 목록 출력 (pv, pvc 로)
$ kubectl get pv,pvc
```

VOLUME_ID (EBS 생성 시 cell 변수에  저장) 통해 퍼시스턴트 볼륨 생성

```bash
$ cat ebs-pv.yaml | sed "s/<VOLUME_ID>/$VOLUME_ID/g" | kubectl apply -f -
```

볼륨 목록 출력 
(`퍼시스턴트 볼륨`은 **네임스페이스**에 속하지 않는 **클러스터** 단위의 오브젝트, 따라서 해당 명령어는 네임스페이스 상관 없이 모든 퍼시스턴트 볼륨 출력함)

- 참고 - **네임스페이스**와 **클러스터**
    
    쿠버네티스는 클러스터 안에 가상 클러스터를 또 다시 만들 수 있고, **네임스페이스**는 **클러스터** 안의 가상 클러스터를 의미
    * 가상 클러스터는 전체 클러스터의 영역을 각 리소스 용도에 따라 구분하기 위한 논리적인 구분 단위
    * 클러스터 구축 직후 생성되는 네임스페이스 4개 - default, docker, kube-public, kube-system
    
    ```bash
    # 현재 클러스터 안에 존재하는 네임스페이스의 목록 확인
    > kubectl get namespace
    ```
    

```bash
$ kubectl get pv
```

포드와 퍼시스턴트 볼륨 클레임 생성

```bash
$ kubectl apply -f ebs-pod-pvc.yaml

$ kubectl get pv,pvc

$ kubectl get pods
```

포드와 퍼시스턴트 볼륨 클레임 생성, 볼륨 마운트

```bash
$ kubectl apply -f ebs-pod-pvc.yaml

$ kubectl get pv,pvc

$ kubectl get pods

$ kubectl exec ebs-mount-container -- df -h | grep /mnt

```

---

### 9.3.3 퍼시스턴트 볼륨을 선택하기 위한 조건 명시

(yaml 파일 [github](https://github.com/alicek106/start-docker-kubernetes/tree/master/chapter9) 에서 확인)

---

### 9.3.4 퍼시스턴트 볼륨의 라이프사이클과 Reclaim Policy

포드와 퍼시스턴트 볼륨 클레임 생성, 볼륨 마운트

```bash
$ kubectl get pv,pvc

$ kubectl get pv,pvc

$ kubectl delete -f ebs-pod-pvc.yaml

$ kubectl get pv
```

포드와 퍼시스턴트 볼륨 클레임 생성, 볼륨 마운트

```bash
$ kubectl delete -f ebs-pv.yaml

$ cat ebs-pv.yaml | sed "s/<VOLUME_ID>/$VOLUME_ID/g" | kubectl apply -f -

$ kubectl get pv
```

`ebs-pv-delete.yaml` 파일 이용해  퍼시스턴트 볼륨 생성

```bash
$ kubectl delete -f ebs-pv.yaml

$ cat ebs-pv.yaml | sed "s/<VOLUME_ID>/$VOLUME_ID/g" | kubectl apply -f -

$ kubectl get pv
```

Reclaim Policy 가 Delete 인 퍼시스턴트 볼륨을 포드 내부에 마운트한 뒤, 연결된 퍼시스턴트 볼륨 클레임을 삭제함으로써 사용 종료

```bash
$ kubectl apply -f ebs-pod-pvc.yaml

$ kubectl get pods

$ kubectl get pv,pvc

$ kubectl delete -f ebs-pod-pvc.yaml

$ kubectl get pv,pvc
```

---

### 9.3.5 StorageClass와 Dynamic Provisioning

AWS에서 다이나믹 프로비저닝 사용하기

```bash
# 생성되어 있는 퍼시스턴트 볼륨과 포드 전부 삭제
$ kubectl delete pv,pvc,pods,deployment --all

# 스토리지 클래스 목록 확인
$ kubectl get storageclass
$ kubectl get sc

# 스토리지 클래스 생성
$ kubectl apply -f storageclass-slow.yaml
$ kubectl apply -f storageclass-fast.yaml
$ kubectl get sc

# 위 스토리지 클래스 사용하는 퍼시스턴트 볼륨 클레임 생성하여 다이나믹 프로비저닝 발생
$ kubectl apply -f pvc-fast-sc.yaml

# 다이나믹 프로비저닝으로 인하여 동적으로 생성된 SSD 타입의 EBS와 그로 인한 퍼시스턴트 볼륨 확인
$ kubectl get pv,pvc
$ kubectl get sc fast -o yaml 
```

다이나믹 프로비저닝에서 특정 스토리지 클래스를 기본값으로 사용 (주석을 사용한 `storagesclass-default.yaml` 파일 활용)

```bash
$ kubectl apply -f storagesclass-default.yaml
$ kubectl get storagesclass
```

리소스 정리

```bash
$ cd chapter9/
$ kubectl delete -f ./
```
