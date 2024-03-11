# MetalLB

## MetalLB 란?

- Bare Metal Loadbalancer 의 약자이다.
- 온프레미스 환경에서 LoadBalancer 서비스를 사용하기 위한 솔루션
- Layer 2, BGP 모드 지원
- 쿠버네티스 클러스터 내부에서 로드 밸런서 역할을 수행가능
  - 서비스(로드 밸런서)의 External IP 를 사용하여 외부에서 접근 가능
  - ARP, BGP를 사용하여 ip 주소를 할당하고 관리
  - 데몬셋으로 speaker 파드를 생성하여 External IP 를 전파

## metalLB 설치

**반드시 metallb-system 네임스페이스에 설치해야 함 - 권장됨**

1. helm 설치 및 레포지토리 추가

```bash
brew install helm
helm repo add metallb https://metallb.github.io/metallb
```

2. kube-system 네임스페이스의 kube-proxy config 설정 변경

```bash
kubectl edit configmap -n kube-system kube-proxy
```

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  # 여기가 변경된 부분 쿠버네티스 1.18 버전 이상에서는 true 로 설정해야 함
  strictARP: true 
```

3. metallb 설치

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
kubectl create namespace metallb-system
helm install metallb metallb/metallb -n metallb-system
```

4. metallb 설치 확인

```bash
kubectl get all -n metallb-system
```

## metalLB 설정

자세한 설정 방법은 [여기](https://metallb.universe.tf/configuration/)를 참고 </br>
자세한 설정 예제는 [여기](https://github.com/metallb/metallb/tree/v0.14.3/configsamples)를 참고 </br>
모든 API 는 설명은 [여기](https://metallb.universe.tf/apis/)를 참고

1. ip pool 설정

```yaml
# ip-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.49.0/24
#   - 192.168.49.240-192.168.49.250 # 이 방법도 가능
```

```bash
kubectl apply -f ip-pool.yaml
```

2. L2 Advertisement 설정

```yaml
# l2-advertisement.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: first-metallb-l2
  namespace: metallb-system
spec:
  # 이걸 정의하지 않으면 모든 ip pool 에 대해 advertisement 를 수행
#   ipAddressPools: 
#   - first-ip-pool
```

3. [보류] BGP 설정 -  BGP 모드를 사용할 때 설정

4. 설정 확인

```bash
kubectl get IPAddressPool -n metallb-system
kubectl get L2Advertisement -n metallb-system
```

## metalLB 사용 예제

이제 쿠버네티스에서 service를 생성할 때 LoadBalancer 타입을 사용하여 외부에서 접근 가능한 서비스를 생성할 수 있다.
```spec.type: LoadBalancer``` 를 사용하면 metallb 가 자동으로 External IP 를 할당해준다. <br>
만약, 오류가 발생한다면 ```kubectl describe svc <svc-name>``` 명령어를 사용하여 확인해보자.

1. 지정된 ip로 서비스 생성

설정된 ip pool 중에서 지정된 ip 로 서비스를 생성할 수 있다. </br>
그러나, 이 방법은 ip pool 에서 지정된 ip 가 다른 서비스에서 사용중이라면 오류가 발생한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.1.100
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

2. 자동으로 ip 할당되는 서비스 생성

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    metallb.universe.tf/address-pool: first-ip-pool # 내가 설정한 ip pool 이름
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

3. 전체 예제 참고 자료 <br>
링크: [여기](https://metallb.universe.tf/usage/example/)


## metalLB 삭제
```bash
helm uninstall metallb -n metallb-system
kubectl delete namespace metallb-system
```
