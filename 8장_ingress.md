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

```

# 8.3 인그레스의 세부 기능: annotation을 이용한 설정

```bash

```

# 8.4 Nginx 인그레스 컨트롤러에 SSL/TLS 보안 연결 적용

```bash

```

# 8.5 여러 개의 인그레스 컨트롤러 사용하기

```bash

```
