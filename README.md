# Model
## azure-aks-autoscaling-and-high-availability-lab-20250618
https://labs.msaez.io/#/189596125/storming/modelforops2

Kubernetes 애플리케이션 자동 확장 (Auto-Scaling) 및 고가용성 실습
- 이 실습은 쿠버네티스에서 애플리케이션의 Scale-Out 방식을 학습합니다.
- 수동 스케일링부터 Horizontal Pod Autoscaler (HPA)를 이용한 자동 스케일링까지 구현하며, Metric Server 및 Pod의 Resource 정의의 중요성을 이해합니다.
- 이를 통해 동적으로 변화하는 트래픽에 대응하는 탄력적인 서비스 운영 능력을 기릅니다.

![스크린샷 2025-06-18 150222](https://github.com/user-attachments/assets/69ef9123-7177-4052-a972-dbce2cd359b8)
![스크린샷 2025-06-18 150516](https://github.com/user-attachments/assets/2a8c1fa2-0b44-4161-8892-afd4be6db31a)
![스크린샷 2025-06-18 151209](https://github.com/user-attachments/assets/c2ff87b3-679a-447d-b066-69fb06891e2d)
![스크린샷 2025-06-18 151733](https://github.com/user-attachments/assets/2eb0bde1-d692-4de4-8365-7c4cf34e145b)
![스크린샷 2025-06-18 153552](https://github.com/user-attachments/assets/8cb64259-933f-4d35-ae5f-f6d2fbf702a8)
![스크린샷 2025-06-18 154744](https://github.com/user-attachments/assets/4efbfdba-5e24-48d4-bdc9-645c24a2a299)
![스크린샷 2025-06-18 155027](https://github.com/user-attachments/assets/c7c45dd4-7289-44a4-b6a9-572f30aac05b)

## 실습 단계별 상세 설명

0. 환경 초기화
```
# 환경 초기화 스크립트 실행 (init.sh 스크립트가 존재한다고 가정)
./init.sh
```
1. 이전 실습 리소스 정리
목표: 이전 실습에서 배포되었을 수 있는 home Deployment를 깔끔하게 삭제하여 현재 실습 환경을 준비합니다.
```
# 'home' Deployment 삭제
kubectl delete deploy home
```
2. 수동 스케일 아웃 (Manual Scale-Out) 실습
목표: 주문 서비스 (order)를 배포하고 서비스로 노출한 후, 수동으로 Pod 개수를 조정하여 ReplicaSet의 기본적인 확장 및 축소 동작을 이해합니다.
```
# 주문 서비스 Deployment 배포
kubectl create deploy order --image=jinyoung/monolith-order:v20210504

# 배포된 주문 서비스를 ClusterIP 타입의 Service로 노출 (8080 포트)
kubectl expose deploy order --port=8080

# 'order' Deployment의 Pod 개수를 3개로 수동 확장
kubectl scale deploy order --replicas=3

# 모든 Kubernetes 리소스 상태 확인 (확장된 Pod 확인)
kubectl get all

# 'order' Deployment의 Pod 개수를 1개로 수동 축소
kubectl scale deploy order --replicas=1

# 모든 Kubernetes 리소스 상태 재확인 (축소된 Pod 확인)
kubectl get all
```
3. 자동 스케일 아웃 준비: 부하 테스트 Pod 설치
목표: 자동 스케일링(Auto-Scaling) 동작을 트리거하기 위해 서비스에 인위적인 부하를 발생시킬 siege 워크로드 생성기 Pod를 배포하고 테스트합니다.
```
# 'siege' 부하 테스트용 Pod 배포
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: siege
spec:
  containers:
  - name: siege
    image: apexacme/siege-nginx
EOF

# 생성된 'siege' Pod 내부 터미널로 접속
kubectl exec -it siege -- /bin/bash

# siege 터미널에서 주문 서비스에 소규모 부하 테스트 (1개 동시 사용자, 2초간 실행)
siege -c1 -t2S -v http://order:8080/orders

# siege 터미널 종료
exit
```
4. Metric Server 설치 확인 및 배포
목표: Kubernetes Horizontal Pod Autoscaler (HPA)가 Pod의 CPU/메모리 사용량과 같은 메트릭을 기반으로 자동 확장 결정을 내리기 위해서는 Metric Server가 클러스터에 필수적으로 설치되어 있어야 합니다. kubectl top pods 명령을 통해 설치 여부를 확인하고, 설치되어 있지 않다면 공식 YAML 파일을 통해 배포합니다.
```
# Pod들의 CPU/메모리 사용량 확인 (Metric Server 설치 여부 확인)
kubectl top pods

# "error: Metrics API not available" 메시지가 나오면 Metric Server 설치
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Metric Server Deployment 상태 확인 (kube-system 네임스페이스)
kubectl get deployment metrics-server -n kube-system
```
5. 주문 서비스에 자동 스케일링(HPA) 설정
목표: Horizontal Pod Autoscaler (HPA)를 설정하여 order Deployment가 Pod의 CPU 사용률에 따라 자동으로 확장 및 축소되도록 구성합니다. CPU 사용률이 50%를 초과하면 Pod 수가 늘어나도록 하며, 최소 1개에서 최대 3개까지 Pod가 확장되도록 제한을 둡니다.
```
# 'order' Deployment에 HPA 설정: CPU 사용률 50% 초과 시 확장, 최소 1개, 최대 3개 Pod 유지
kubectl autoscale deployment order --cpu-percent=50 --min=1 --max=3
참고: HPA가 메트릭을 수집하고 동작하기까지 약간의 시간이 소요될 수 있습니다. HPA의 상태는 kubectl get hpa 명령으로 확인할 수 있습니다.
```
6. order-deploy.yaml 생성 및 리소스 정의
목표: 자동 스케일링이 정확하게 동작하고 Pod의 안정적인 운영을 보장하기 위해 컨테이너의 CPU/메모리 requests (요청량) 및 limits (제한량)를 명시적으로 정의합니다. 또한 readinessProbe (트래픽 수용 준비 상태)와 livenessProbe (애플리케이션 생존 상태)를 정의하여 Kubernetes가 Pod의 건강 상태를 정확히 인지하도록 설정합니다.
```
order-deploy.yaml 파일을 생성하고 다음 내용을 복사하여 저장합니다:

YAML

# order-deploy.yaml 파일 내용
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order
  labels:
    app: order
spec:
  replicas: 1 # 초기 Pod 개수
  selector:
    matchLabels:
      app: order
  template:
    metadata:
      labels:
        app: order
    spec:
      containers:
        - name: order # 컨테이너 이름
          image: jinyoung/monolith-order:v20210602 # 새로운 이미지 버전
          ports:
            - containerPort: 8080
          resources: # Pod가 요청하고 사용할 수 있는 리소스 정의
            requests:
              cpu: "200m" # 0.2 CPU 코어 요청 (1000m = 1 CPU 코어)
              memory: "256Mi" # 256 MiB 메모리 요청
            limits:
              cpu: "500m" # 최대 0.5 CPU 코어까지 사용 제한 (초과 시 스로틀링)
              memory: "512Mi" # 최대 512 MiB 메모리 사용 제한
          readinessProbe: # Pod가 네트워크 트래픽을 받을 준비가 되었는지 확인
            httpGet:
              path: '/actuator/health' # 헬스 체크 엔드포인트
              port: 8080
            initialDelaySeconds: 10 # Pod 시작 후 첫 프로브까지 대기 시간
            timeoutSeconds: 2 # 프로브 타임아웃 시간
            periodSeconds: 5 # 프로브 주기
            failureThreshold: 10 # 실패 횟수 임계치 (Pod NotReady)
          livenessProbe: # Pod가 정상적으로 동작 중인지 (kill 여부 결정) 확인
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 120 # Pod 시작 후 첫 프로브까지 대기 시간
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5 # 실패 횟수 임계치 (Pod 재시작)
설명: resources.requests.cpu: "200m"은 해당 컨테이너가 최소 0.2 CPU 코어를 요청한다는 의미입니다. HPA는 이 requests 값을 기준으로 CPU 사용률을 계산합니다 (예: 0.1 CPU 사용 시, 200m 중 100m 사용이므로 50% 사용률). limits는 컨테이너가 최대로 사용할 수 있는 리소스를 제한하여 다른 Pod에 영향을 주지 않도록 합니다. 프로브는 Pod의 안정적인 운영에 필수적입니다.
```
7. order Deployment 삭제 후 재배포
목표: 새로운 resources 및 probes 설정이 적용된 order-deploy.yaml 파일을 클러스터에 반영하기 위해 기존 Deployment를 삭제하고 새 YAML 파일로 다시 배포합니다.
```
# 기존 'order' Deployment 삭제
kubectl delete deploy order

# 리소스 정의가 포함된 'order-deploy.yaml' 파일로 'order' Deployment 재배포
kubectl apply -f order-deploy.yaml

# 배포된 'order' Deployment의 상세 정보 확인 (특히 리소스 요청/제한 확인)
kubectl get deploy order -o yaml
참고: kubectl get hpa 명령을 다시 실행하면 CPU(%) 컬럼에 이제 3/50%와 같이 리소스 사용률이 표시될 것입니다. 이는 HPA가 Pod의 CPU 사용량을 정확히 모니터링하기 시작했다는 의미입니다.
```
8. 부하 발생 및 자동 확장 관찰
목표: siege Pod를 통해 order 서비스에 높은 부하를 발생시키고, Horizontal Pod Autoscaler (HPA)가 설정된 CPU 임계치(50%)를 초과할 때 order Pod가 자동으로 확장되는 과정을 실시간으로 관찰합니다.
```
# 'siege' Pod 내부 터미널로 접속
kubectl exec -it siege -- /bin/bash

# siege 터미널에서 'order' 서비스에 고강도 부하 발생 (20개 동시 사용자, 40초간 실행)
siege -c20 -t40S -v http://order:8080/orders

# siege 터미널 종료
exit

# HPA 상태를 지속적으로 모니터링 (새 터미널에서 실행 권장)
# kubectl get hpa -w
# Pod 개수 변화 모니터링 (새 터미널에서 실행 권장)
# watch kubectl get pods -l app=order
설명: resources.requests.cpu: "200m"을 100% 기준으로 했을 때, Pod가 1200m (1.2 CPU 코어)를 사용한다면 이는 600%의 CPU 사용률로 인식됩니다. Kubernetes 클러스터의 노드는 여러 CPU 코어를 가지고 있으므로, Pod의 리소스 요청 및 제한은 클러스터의 전체 리소스 사용 효율성과 안정성에 중요합니다. 리소스 설정값은 HPA의 스케일링 기준이 되며, 실제 Pod의 성능은 워크로드와 클러스터 상황에 따라 유연하게 조절될 수 있습니다.
```
