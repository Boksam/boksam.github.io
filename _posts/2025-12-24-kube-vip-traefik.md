---
title: "K3s 환경에서 Kube-vip를 로드밸런서 컨트롤러로 활용하기"
date: 2025-12-24
categories: [Infrastructure, Kubernetes]
tags:
  - Kubernetes
  - Kube-vip
  - Traefik
  - TLS
  - Code Place
image: "/assets/2025-12-24-kube-vip-traefik/traefik-logo.jpg"
---

> 이 포스트에서는 K3s의 내장 로드밸런서인 ServiceLB 대신 Kube-vip를 사용하여 Traefik 서비스에 External IP를 할당하는 방법을 정리합니다.
{: .prompt-info}

## 1. 개요

최근 [코드플레이스](https://code.pusan.ac.kr) 프로젝트를 Kubernetes 환경으로 마이그레이션하면서 네트워크 인프라의 가용성을 높이는 작업을 진행하고 있습니다.

이전 글인 [Kube-vip 기반 고가용성 K3s 클러스터 구축]({{ site.baseurl }}{% link _posts/2025-12-13-kube-vip.md %})에서 다룬 것처럼, 저는 `Kube-vip`를 사용하여 HA 클러스터를 구성했습니다.
이번 글에서는 Kube-vip를 단순히 Control Plane의 고가용성 확보에 그치지 않고,
로드밸런서 컨트롤러로 설정하여 `Traefik`이 외부 트래픽을 안정적으로 수용할 수 있도록 구성한 과정을 공유하고자 합니다.

## 2. K3s의 기본 로드밸런서 컨트롤러: ServiceLB

K3s를 설치하면 기본적으로 `ServiceLB`라는 내장 로드밸런서 컨트롤러가 활성화됩니다.

ServiceLB는 별도의 외부 로드밸런서 장비가 없는 환경에서, 클러스터 노드의 IP를 활용해 외부 트래픽을 서비스로 유입시키는 역할을 합니다.
`LoadBalancer` 타입의 서비스가 생성되면 노드의 포트를 점유하여 트래픽을 포워딩해주는 방식입니다.

### 2-1. ServiceLB 비활성화

하지만 저는 K3s 설치 단계에서 `--disable servicelb` 옵션을 사용하여 이를 비활성화했습니다. 그 이유는 다음과 같습니다.

1. **엔드포인트의 단일화 (고정 VIP)**: 코드플레이스 프로젝트는 가상 IP(VIP)를 통해 모든 트래픽을 수용하도록 설계되었습니다. 만약 ServiceLB를 켜두면 클러스터 노드들의 실제 물리 IP가 서비스에 할당될 수 있는데, 이는 VIP를 이용해 Fault Tolerance를 확보하려는 구성 의도와 맞지 않습니다.
2. **컨트롤러 간의 충돌 방지**: 고가용성(HA)을 위해 이미 `Kube-vip`를 사용 중인 상황에서 ServiceLB까지 활성화되면, 두 컨트롤러가 하나의 서비스 객체를 두고 서로 IP를 할당하려다 충돌이 발생할 수 있습니다. 따라서 Kube-vip가 로드밸런서 컨트롤러 역할까지 전담하도록 역할을 단일화했습니다.

ServiceLB를 비활성화 한 상태로 Traefik을 배포하면, IP를 할당해 줄 주체가 없으므로 아래와 같이 Traefik Service의 EXTERNAL IP 가 `<pending>` 상태에 머물게 됩니다.

```sh
$ kubectl get svc -n kube-system
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
traefik   LoadBalancer   10.43.176.83    <pending>     80:30080/TCP,443:30443/TCP   10m
```

### 2-2. External IP 가 없으면 외부에서 접속할 수 없는 이유

Kubernetes 서비스는 기본적으로 클러스터 내부에서만 유효한 IP인 **ClusterIP** 를 가집니다.
하지만 우리가 브라우저에 주소를 입력해서 접속하려면 외부에서 접근할 수 있는 IP가 필요합니다.

- **외부-내부 통로 없음**: External IP가 `<pending>`이라는 것은, 외부와 내부 간의 통로가 없다는 뜻입니다.
- **라우팅 불가능**: 외부 사용자가 내부 IP를 통해 접근하려고 해도, 인터넷 상의 라우터들은 해당 요청을 어디로 보내야 할지 알 수 없습니다.

![](/assets/2025-12-24-kube-vip-traefik/traefik-no-external-ip.png)
_External IP가 없는경우 외부 접근 불가_

따라서 External IP가 할당되지 않으면, 외부에서 Traefik을 통해 서비스에 접속할 수 없게 되므로, 이 Traefik 서비스가 External IP를 가져야만 외부에서 접근이 가능해집니다.

## 3. Kube-vip를 이용한 로드밸런서 설정

### 3-1. Kube-vip의 External IP 할당 방식
이제 Kube-vip가 ServiceLB 대신 로드밸런서 컨트롤러 역할을 할 수 있도록 만들어야 합니다.
Kube-vip가 로드밸런서 컨트롤러로서 External IP를 할당하는 방식은 크게 두 가지가 있습니다.

| 방식 | 특징 | 활용 사례 |
| --- | --- | --- |
| **Annotation** | 서비스 매니페스트에 특정 IP를 Annotation으로 명시 | 특정 서비스에 고정 IP 부여 시 |
| **IP Pool** | 정의된 IP 대역에서 남는 IP를 자동 할당 | 다수의 서비스를 운영하며 자동화가 필요할 때 |

코드플레이스 프로젝트에서는 모든 외부 트래픽이 사전에 정의된 **가상 IP(VIP)**로 들어와야 했습니다. 따라서 저는 Annotation 방식을 선택했습니다.

### 3-2. Traefik에 가상 IP 할당

K3s에 내장된 Traefik의 설정을 변경하려면 직접 Service 객체를 수정하기보다 HelmChartConfig 리소스를 사용하는 것이 좋습니다.
`/var/lib/rancher/k3s/server/manifests/traefik-config.yaml` 파일을 생성하고 다음과 같이 작성합니다.

- Kube-vip 0.5.12 이상: `metadata.annotations.kube-vip.io/loadBalancerIPs`에 작성
- Kube-vip 0.5.11 이하: `spec.loadBalancerIP`에 작성

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    service:
      enabled: true
      type: LoadBalancer
      annotations:
        kube-vip.io/loadbalancerIPs: <가상 IP>
```

> `/var/lib/rancher/k3s/server/manifests/traefik.yaml`은 K3s 실행될때마다 자동으로 초기화되므로,
> `traefik-config.yaml` 파일을 별도로 만들어 HelmChartConfig를 통해 설정을 덮어쓰는 방식을 사용해야 합니다.
{: .prompt-tip}

## 4. 결과 확인

`/var/lib/rancher/k3s/server/manifests/`에 Traefik의 HelmChartConfig를 생성하고 나면 K3s가 자동으로 적용합니다.
잠시 기다리면, Traefik 파드가 재실행되고 서비스에 External IP가 할당된 것을 확인할 수 있습니다.

```sh
$ kubectl get svc -n kube-system
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
traefik   LoadBalancer   10.43.176.83    가상 IP        80:30080/TCP,443:30443/TCP   10m
```

아래와 같이 `dev` 환경에서 Ingress를 생성하고 `k3s.code-place-dev.site`와 연결해두었는데요, 브라우저에서 정상적으로 연결되는지 확인해보겠습니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.tls.certresolver: default
spec:
  rules:
    - host: k3s.code-place-dev.site
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
  tls:
    - hosts:
        - k3s.code-place-dev.site
```

![](/assets/2025-12-24-kube-vip-traefik/dev-web-result.png)
_k3s.code-place-dev.site 접속 결과_

## 5. 마치며

이번에 Kube-vip + Traefik을 구성해보면서 K3s의 네트워크 동작 원리를 한 층 더 깊게 이해할 수 있게 되었습니다.

특히 서비스 가용성을 위해 도입한 `Kube-vip`가 Control Plane의 고가용성뿐만 아니라 서비스 로드밸런싱까지 책임지게 함으로써 인프라의 복잡성을 줄이고 관리 포인트를 일원화할 수 있었던 점과,
외부에서 클러스터 내부로 접근할 수 있는 External IP 원리에 대해 공부할 수 있었던 점이 좋았습니다.

베어메탈 환경에서 특정 가상 IP를 통해 트래픽 유입 경로를 단일화하고 싶은 분들께 이 기록이 도움이 되길 바랍니다.

---

## References

- [K3s Networking Services](https://docs.k3s.io/networking/networking-services)
- [Kubernetes Load-Balancer service](https://kube-vip.io/docs/usage/kubernetes-services/)
