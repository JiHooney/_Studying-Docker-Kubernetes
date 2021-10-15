```bash
인그레스는 일반적으로 외부에서 내부로 향하는 것을 의미한다. 
예를 들어 인그레스 트래픽은 외부에서 서버로 유입되는 트래픽을 의미하며,
인그레스 네트워크는 인그레스 트래픽을 처리하기 위한 네트워크를 의미한다.

이전에 사용했던 서비스 오브젝트가 외부 요청을 받아들이기 위한 것이라면
인그레스는 외부 요청을 어떻게 처리할 것인지 네트워크 7계층 레벨에서 정의하는 
쿠버네티스 오브젝트이다.

*** 인그레스 오브젝트의 기본 기능**
-**외부 요청의 라우팅**: /apple, /apple/red 등과 같이 특정 경로로 들어온 요청을 어떠한 서비스로
  전달할지 정의하는 라우팅 규칙을 설정한다.
-**가상 호스트 기반의 요청 처리**: 같은 IP에 대해 다른 도메인 이름으로 요청이 도착했을 때, 
 어떻게 처리할 것인지 정의할 수 있다.
**-SSL/TLS 보안연결 처리**: 여러 개의 서비스로 요청을 라우팅할 때, 보안 연결을 위한 인증서를 
 쉽게 적용할 수 있다.

```

# 8.1 인그레스를 사용하는 이유

```bash
인그레스를 사용하는 이유는 다음과 같다.

예를 들어 애플리케이션이 3개의 디플로이먼트로 생성돼 있다고 가정한다.
각 디플로이먼트를 외부에 노출해야 한다면 NodePort 또는 LoadBalancer 타입의 서비스 3개를
생성하는 방법이 있다.
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d195e2dd-7570-44d4-b5ab-99fea1009071/Untitled.png)

```bash
위 방식에서 서비스마다 세부적인 설정을 할 때 복잡해진다. SSL/TLS 보안 연결, 접근 도메인 및
클라이언트 상태에 기반한 라우팅 등을 구현하려면 각 서비스와 디플로이먼트에 대해 일일이
설정을 해야 하기 때문이다. 

이때 쿠버네티스가 제공하는 인그레스 오브젝트를 사용하면 URL 엔드포인트를 한 하나만
생성함으로써 이러한 번거로움을 쉽게 해결할 수 있다. 
3개의 디플로이먼트를 외부로 노출하는 인그레스를 생성하면 다음과 같다.
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c9f323e9-a15e-451a-ac3b-d49ba67cf875/Untitled.png)

```bash
위 그림에서 3개의 서비스에 대해 3개의 URL이 각각 존재하는 것이 아닌,
인그레스에 접근하기 위한 단 하나의 URL만 존재한다. 
따라서 클라이언트는 인그레스의 URL로만 접근하게 되며,
해당 요청은 인그레스에서 정의한 규칙에 따라 처리된 뒤 적절한 디플로이먼트의
포드로 전달된다.
```

# 8.2 인그레스의 구조

```bash
인그레스는 쿠버네티스에서 ingress라는 이름으로 사용할 수 있고,
kubectl get ingress 명령어로 인그레스의 목록을 확인할 수 있다.

> kubectl get ingress
> kubectl get ing

#인그레스 생성하기
> vi chapter8/ingress-example.yaml

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: alicek106.example.com                  
    http:
      paths:
      - path: /echo-hostname                     
        backend:
          serviceName: hostname-service          
          servicePort: 80

#인그레스 생성
> kubectl apply -f ingress-example.yaml

#인그레스 목록 확인
> kubectl get ingress

인그레스를 생성하는 것만으로는 아무 일도 일어나지 않는다.
인그레스는 단지 외부 요청을 처리하는 규칙을 정의하는 오브젝트일 뿐,
외부 요청을 받아들일 수 있는 실제 서버가 아니기 때문이다.

인그레스는 **인그레스 컨트롤러(Ingress Controller)라**고 하는 특수한 서버에 적용해야지
그 규칙을 사용할 수 있다.

**즉, 실제로 외부 요청을 받아들이는 것은 인그레스 컨트롤러 서버이며,** 
이 서버가 인그레스 규칙을 로드해 사용한다.
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e6af87f9-420a-4b48-be56-1fa099b30ef9/Untitled.png)

```bash
따라서 쿠버네티스의 인그레스는 반드시 인그레스 컨트롤러라는 서버와 함께 사용해야 한다.
인그레스 컨트롤러 서버는 여러 종류가 있으며, 필요에 따라 하나를 골라서 사용하면 된다.

대표적으로는 Nginx 웹 서버 인그레스 컨트롤러가 있다.

아래의 명령어를 실행하면 Nginx 인그레스 컨트롤러와 관련된 모든 리소스를 한 번에 설치할 수 있다.

> kubectl apply -f \
https://raw.githubsercontent.com/kubernetes/ingress-nginx/controller-v0.35.0/deploy/static/provider/aws/deploy.yaml

실행결과를 보면 Nginx 인그레스 컨트롤러를 설치하기 위해 다양한 쿠버네티스 리소스를 한 번에 생성한다.
우선 ingress-nginx라는 네임스페이스에 Nginx 웹 서버 디플로이먼트를 생성하고,
그와 관련된 설정들을 컨피그맵으로 생성한다.

ingress-nginx 네임스페이스의 디플로이먼트와 포드를 확인해보면 Nginx 웹서버가 생성되어 있음을
확인할 수 있다.
> kubectl get pods,deployment -n ingress-nginx

외부에서 Nginx 인그레스 컨트롤러에 접근하기 위한 서비스도 생성됐다.
> kubectl get svc -n ingress-nginx

Nginx 인그레스 컨트롤러를 설치하면 자동으로 생성되는 서비스는 LoadBalancer 타입이다.
실제 운영환경이라면 LoadBalancer 타입에 DNS이름을 할당함으로써 Nginx 인그레스 컨트롤러에
접근하는 것이 일반적이지만 여기서는 자동으로 부여된 DNS이름을 사용해본다.
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ed26ae6e-578d-4e64-91dd-1c4cba5e2f5f/Untitled.png)

```bash
> vi ingress-nginx-svc-nodeport.yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
  name: ingress-nginx-controller-nodeport
  namespace: ingress-nginx
spec:
  ports:
  - name: http
    nodePort: 31000
    port: 80
    protocol: TCP
    targetPort: http
  - name: https
    nodePort: 32000
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: NodePort

이렇게 해서 인그레스, Nginx 인그레스 컨트롤러 및 Nginx 포드에 접근하기 위한 서비스의
준비가 완료됐다. 이제 인그레스의 종착점이 될 테스트용 디플로이먼트와 서비스를 생성해본다.

> vi hostname-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostname-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver
  template:
    metadata:
      name: my-webserver
      labels:
        app: webserver
    spec:
      containers:
      - name: my-webserver
        image: alicek106/ingress-annotation-test:0.0
        ports:
        - containerPort: 5000
          name: flask-port

> kubectl apply -f hostname-deployment.yaml

> vi hostname-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: hostname-service
spec:
  ports:
    - name: web-port
      port: 80
      targetPort: flask-port
  selector:
    app: webserver
  type: ClusterIP

> kubectl apply -f hostname-service.yaml

#포드와 서비스 목록 확인
> kubectl get pods,services

만약 AWS를 사용하고 있다면 DNS이름으로 요청을 전송해본다.
> curl <AWS의 DNA이름>/echo-hostname

이때 404에러가 뜨는 이유는 이전에 인그레스를 생성할 때 Nginx 인그레스 컨트롤러에
alicek106.example.com으로 접근했을 때만 응답을 처리하도록 설정했기 때문이다.
따라서 alicek106.example.com이 아닌 다른 도메인 이름으로 접근할 때는 Nginx인그레스
컨트롤러가 해당 요청을 처리하지 않는다.

Nginx에 접근하기 위한 LoadBalancer 타입의 서비스를 생성했을 때
curl 명령어의 --resolve 옵션을 통해 임시로 도메인명을 설정할 수 있다.

> curl --resolve alicek106.example.com:80:<부하분산기IP> alicek106.example.com/echo-hostname

#사용 예시
> curl --resolve alicek106.example.com:80:1.2.3.4 alicek106.example.com/echo-hostname

NodePort 타입으로 서비스를 생성했다면 다음과 같이 테스트할 수 있다.
> curl --resolve alicek106.example.com:31000:<노드 중 하나의 IP> \
alicek106.example.com:31000/echo-hostname

AWS에서 임의의 DNS를 LoadBalancer타입의 서비스에 할당받았다면 이미 생성된 인그레스 리소스의
정보에서 host 항목을 직접 로드밸런서의 DNS로 수정해본다.

> kubectl edit ingress ingress-example

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: <서비스의 DNS명 입력>           
    http:
      paths:
      - path: /echo-hostname                     
        backend:
          serviceName: hostname-service          
          servicePort: 80

인그레스의 host항목을 변경했기 때문에 AWS로드밸런서의 DNS 이름으로 접근할 때도
Nginx 인그레스 컨트롤러가 해당 요청을 처리한다.

> curl <AWS의 DNA이름>/echo-hostname
```

- 인그레스 컨트롤러의 동작 원리 이해

```bash
인그레스를 사용하는 방법
1. 공식 깃허브에서 제공되는 YAML 파일로 Nginx 인그레스 컨트롤러를 생성한다.
2. Nginx 인그레스 컨트롤러를 외부로 노출하기 위한 서비스를 생성한다.
3. 요청 처리 규칙을 정의하는 인그레스 오브젝트를 생성한다.
4. Nginx 인그레스 컨트롤러로 들어온 요청은 인그레스 규칙에 따라 적절한 서비스로 전달된다.

위 과정 중 3번에서 인그레스를 생성하면 인그레스 컨트롤러는 자동으로 인그레스를 로드해 Nginx 웹 서버에 적용한다.
이를 위해 Nginx 인그레스 컨트롤러는 항상 인그레스 리소스의 상태를 지켜보고 있으며, 
기본적으로 모든 네임스페이스의 인그레스 리소스를 읽어와 규칙을 적용한다.

===* 참고
쿠버네티스의 API에는 특정 오브젝트의 상태가 변화하는 것을 확인할 수 있는 Watch라는 API가 있으며,
인그레스 컨트롤러 또한 인그레스 리소스에 대해 Watch API를 사용한다. 
===

특정 경로와 호스트 이름으로 들어온 요청은 인그레스에 정의된 규칙에 따라 서비스로 전달된다.
이전에 생성했던 테스트용 인그레스에서는 /echo-hostname이라는 경로로 들어온 요청을 hostname-service라는 서비스의 80포트로 전달했다.

하지만 요청이 실제로 hostname-service라는 서비스로 전달되는 것은 아니며, **Nginx 인그레스 컨트롤러는 서비스에 의해 생성된 엔드포인트로 요청을 직접 전달한다.**
**즉, 서비스의 ClusterIP가 아닌 엔드포인트의 실제 종착 지점들로 요청이 전달되는 셈이다.** 
이러한 동작을 쿠버네티스에서는 **바이패스(bypass)**라고 한다. 서비스를 거치지 않고 포드로 직접 요청이 전달되기 때문이다.

#endpoints 항목에 출려된 지점으로 요청이 전달된다.
> kubectl get endpoints

```

# 8.3 인그레스의 세부 기능: annotation을 이용한 설정

```bash
인그레스는 YAML 파일의 주석(annotation)항목을 정의함으로써 다양한 옵션을 사용할 수 있다.
이전에 사용했던 두 개의 주석항목을 다시 살펴본다.

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: alicek106.example.com                   
    http:
      paths:
      - path: /echo-hostname                   
        backend:
          serviceName: hostname-service        
          servicePort: 80

kubernetes.io/ingress.class는 해당 인그레스 규칙을 어떤 인그레스 컨트롤러에 적용할 것인지를 의미한다.
인그레스 컨트롤러 서버는 Nginx 외에도 Kong이나 GKE 등 여러 가지 중 하나를 선택해 사용할 수 있다.

그런데 쿠버네티스 클러스터 자체에서 기본적으로 사용하도록 설정된 인그레스 컨트롤러가 존재하는 경우가 있는데,
이 경우에는 어떤 인그레스 컨트롤러를 사용할 것인지 반드시 인그레스에 명시해주는 것이 좋다.

nginx.ingress.kubernetes.io/rewrite-target이라는 주석은 Nginx 인그레스 컨트롤러에서만 사용할 수 있는 기능이다.
이 주석은 인그레스에 정의된 경로로 들어오는 요청을 rewrite-target에 설정된 경로로 전달한다.
예를 들어 Nginx 인그레스 컨트롤러로 /echo-hostname으로 접근하면 hostname-service에는 / 경로로 전달된다.

단, rewrite-target은 /echo-hostname이라는 경로로 시작하는 모든 요청을 hostname-service의 /로 전달한다.
예를 들어 /echo-hostname/alice/bob이라는 경로로 요청을 보내도 똑같이 / 로 전달된다.

> curl http://<AWS주소>/echo-hostname/alice/bob

사실 rewrite-target은 Nginx의 캡처 그룹(Captured groups)과 함께 사용할 때 유용한 기능이다.
캡처 그룹이란 정규 표현식의 형태로 요청 경로 등의 값을 변수로서 사용할 수 있는 방법이다.

> vi ingress-rewrite-target.yaml

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2 # path의 (.*) 에서 획득한 경로로 전달합니다.
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: <여러분이 Nginx 컨트롤러에 접근하기 위한 도메인 이름을 입력합니다>
  #- host: a2cbfefcfbcfd48f8b4c15039fbb6d0a-1976179327.ap-northeast-2.elb.amazonaws.com
    http:
      paths:
      - path: /echo-hostname(/|$)(.*)          # (.*) 을 통해 경로를 얻습니다.
        backend:
          serviceName: hostname-service
          servicePort: 80

> kubectl apply -f ingress-rewrite-target.yaml

rewrute-target과 path를 수정했다. path 항목에서 (.*)은 Nginx 정규 표현식을 통해 /echo-hostname/ 뒤에 오는 경로를 얻은 뒤, 
이 값을 rewrite-target에서 사용한 것뿐이다.
즉, /echo-hostname/으로 접근하면 이전과 동일하게 /로 전달되지만, /echo-hostname/color는 /color로, /echo-hostname/color/red는 
/color/red로 전달된다.
```

# 8.4 Nginx 인그레스 컨트롤러에 SSL/TLS 보안 연결 적용

```bash
인그레스의 장점 중 하나는 쿠버네티스의 뒤쪽에 있는 디플로이먼트와 서비스가 아닌, 
앞쪽에 있는 인그레스 컨트롤러에서 편리하게 SSL/TLS 보안 연결을 설정할 수 있다.
즉, 인그레스 컨트롤러 지점에서 인증서를 적용해 두면 요청이 전달되는 애플리케이션에 대해 모두 인증서 처리를 할 수 있다.
따라서 인그레스 컨트롤러가 보안 연결을 수립하기 위한 일종의 관문 역할을 한다고도 볼 수 있다.

AWS와 같은 클라우드 환경에서 LoadBalancer 타입의 서비스를 사용할 계획이라면 클라우드 플랫폼 자체에서 
관리해주는 인증서를 인그레스 컨트롤러에 적용할 수도 있다.

#보안 연결에 사용할 인증서와 비밀키 생성하기
이때 "CN=alicek106.~"에는 Nginx 인그레스 컨트롤러에 접근하기 위한 Public DNS 이름을 입력해야 한다.

> openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout tls.key -out tls.crt -subj "/CN=alicek106.example.com/0=alicek106"

#tls타입의 시크릿을 생성
> ls

> kubectl create secret tls tls-secret --key tls.key --cert tls.crt

앞서 만들었던 ingress-example이라는 인그레스의 설정에 TLS 옵션을 추가해서 적용한다.
> vi ingress-tls.yaml

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - alicek106.example.com            # 여러분의 도메인 이름을 입력해야 합니다.
    secretName: tls-secret
  rules:
  - host: alicek106.example.com          # 여러분의 도메인 이름을 입력해야 합니다.
    http:
      paths:
      - path: /echo-hostname
        backend:
          serviceName: hostname-service
          servicePort: 80

```

# 8.5 여러 개의 인그레스 컨트롤러 사용하기

```bash

```
