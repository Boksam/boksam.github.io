---
title: "Kustomize + Github Action으로 이미지 태그 관리 자동화하기"
date: 2026-01-31
categories: [Infrastructure, CI/CD]
tags:
  - Kubernetes
  - Kustomize
  - GitHub Actions
image: "/assets/2026-01-31-kustomize-image-tag/kustomize-logo.png"
---

> Kubernetes 배포 시 `latest` 태그 사용으로 인한 이미지 미갱신 문제를 해결하기 위해,
> GitHub Actions와 Kustomize를 활용하여 이미지 태그를 Git Commit Hash로 자동 업데이트하는 CI 파이프라인 구축 과정을 공유합니다.
{: .prompt-info}

## 1. 개요
---

코드플레이스 서비스를 Kubernetes 환경으로 마이그레이션한 후, 첫 배포 테스트를 진행하던 중 예상치 못한 문제를 마주했습니다.
CI/CD 파이프라인이 성공적으로 돌고 kubectl apply까지 실행되었음에도 불구하고, 실제 파드(Pod)에는 변경된 코드가 반영되지 않는 현상이었습니다.

`kubectl apply -k` 를 실행해보니, 모든 Deployment가 `unchanged`로 표시되었고, 실제로도 이전 버전의 이미지가 계속 사용되고 있었습니다.

```bash
$ kubectl apply -k overlays/dev
...
deployment.apps/backend unchanced
deployment.apps/frontend unchanged
...
```

원인은 **Mutable Tag(가변 태그)**인 latest를 사용했기 때문이었습니다.
Kubernetes의 Deployment는 리소스의 spec이 변경되어야만 롤아웃을 트리거합니다.
하지만 latest라는 태그 이름 자체는 변하지 않았기 때문에, Kubernetes는 이를 '변경 사항 없음(unchanged)'으로 간주하고 새로운 이미지를 풀(Pull)하지 않았던 것입니다.

따라서 저는 Immutable Tag(불변 태그) 전략을 도입하기로 결정했습니다.
매 배포마다 고유한 식별자(Git Commit Hash)를 이미지 태그로 사용하고,
이를 Kustomize 설정에 자동으로 반영하여 확실하게 새로운 이미지를 사용할 수 있도록 함과 동시에 추적 가능한 태그 관리가 가능해지도록 만들고 싶었습니다.

## 2. Kustomize 구조 리팩토링
---

### 2-1. 기존 Kustomize 설정 구조

초기에는 각 환경(dev/prod)의 overlays 디렉토리 내에서 개별 Deployment 패치 파일을 통해 이미지를 관리했습니다.

```
kubernetes/
├── base/
│   ├── frontend-deployment.yaml
│   ├── backend-deployment.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── frontend-deployment-patch.yaml # 여기에 image: ...:latest 정의
    │   ├── backend-deployment-patch.yaml # 여기에 image: ...:latest 정의
    │   └── kustomization.yaml
```

이 구조는 직관적이긴 하지만, 자동화 관점에서는 비효율적이었습니다.
마이크로서비스가 늘어날수록 수정해야 할 파일이 파편화되고, 현재 어떤 버전이 배포되어 있는지 한눈에 파악하기 힘들었기 때문입니다.

### 2-2. `images` 필드를 통한 중앙 관리

Kustomize는 흩어져 있는 이미지 설정을 kustomization.yaml 레벨에서 일괄 관리할 수 있는 [images](https://github.com/openshift/kubernetes-kubectl/blob/master/docs/book/pages/reference/kustomize.md#images) 필드를 제공합니다.
이를 활용해 기존의 복잡한 구조를 단순화하고, 배포 파이프라인이 단 하나의 파일만 바라보도록 리팩토링을 진행했습니다.

먼저, base 디렉토리의 Deployment 파일에서는 구체적인 레지스트리 주소 대신 backend와 같은 단순한 이름을 플레이스홀더(Placeholder)로 지정했습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
# ... (생략)
  template:
    spec:
      containers:
        - name: backend
          # 구체적인 레지스트리 주소 대신 단순한 이름을 사용합니다.
          image: backend:latest 
```
{: file="base/backend-deployment.yaml"}

그다음, 각 환경별(overlays/dev) kustomization.yaml 파일에서 앞서 정의한 backend라는 이름을
실제 사용하는 레지스트리 주소(newName)와 태그(newTag)로 덮어쓰도록 설정했습니다.
기존 패치 파일들에 흩어져 있던 이미지 태그 정의는 모두 제거했습니다.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
# ... (생략)

namespace: code-place-dev

# Deployment 패치 파일에서는 이제 `image` 필드가 제거되어 깔끔해졌습니다.
patches:
  - path: backend-deployment-patch.yaml
  - path: frontend-deployment-patch.yaml

# 모든 이미지 태그를 이 곳에서 중앙 집중적으로 관리합니다.
images:
  - name: backend
    # base에서 지정한 'backend'를 아래 주소로 치환합니다.
    newName: harbor.code-place-dev.site/code-place-dev/backend
    # CI 파이프라인이 이 값을 git commit hash로 업데이트하게 됩니다.
    newTag: eb40822b5b950bf75f00ed053609724a2a10be7f-dev
  - name: frontend
    newName: harbor.code-place-dev.site/code-place-dev/frontend
    newTag: 61af4b605d731c1ec378f124b0e9a44c0438d35b-dev
  - name: judge-server
    newName: harbor.code-place-dev.site/code-place-dev/judge-server
    newTag: 1.0.3
```
{: file="overlays/dev/kustomization.yaml"}

이러한 구조 변경 덕분에 CI 파이프라인은 여러 Deployment 파일을 찾아다닐 필요 없이, 오직 kustomization.yaml 파일 하나만 수정하면 되므로 관리가 훨씬 수월해졌습니다.

## 3. GitHub Actions 워크플로우 구현
---

구현할 전체 파이프라인의 흐름은 다음과 같습니다.

1. 감지: 백엔드나 프론트엔드 코드에 변경이 발생했는지 확인합니다.
2. 빌드: 변경된 서비스의 도커 이미지를 빌드하고, 태그를 Git Commit Hash로 지정하여 푸시합니다.
3. 업데이트 & 커밋: kustomization.yaml의 이미지 태그를 방금 빌드한 Hash 값으로 수정하고, 변경된 Manifest를 Git에 푸시합니다.

![](/assets/2026-01-31-kustomize-image-tag/action-overall-flow.png)
_이미지 태그 자동 업데이트를 위한 GitHub Actions 워크플로우 전체 흐름_

### 3-1. 변경 감지 및 빌드 최적화

불필요한 리소스 낭비를 막기 위해 [dorny/paths-filter](https://github.com/dorny/paths-filter)를 사용하여 실제 코드가 변경된 서비스만 빌드하도록 설정했습니다.
모노레포 구조나 여러 서비스가 함께 있는 저장소에서 불필요한 빌드를 줄이는 데 효과적입니다.

```yaml
detect-changes-by-component:
  runs-on: ubuntu-latest
  outputs:
    backend: ${{ steps.filter.outputs.backend }}
    frontend: ${{ steps.filter.outputs.frontend }}
  steps:
    - uses: actions/checkout@v2
    - uses: dorny/paths-filter@v2
      id: filter
      with:
        filters: |
          backend:
            - 'backend/**'
          frontend:
            - 'frontend/**'
```

이후 빌드 단계(ci-backend-dev)에서는 위에서 감지된 결과(`needs.detect-changes-by-component.outputs.backend == 'true'`)를 조건으로 실행 여부를 결정합니다.
이때 태그는 `${{ github.sha }}-dev` 형식을 사용하여 유일성을 보장했습니다.

### 3-2. Manifest 자동 업데이트 (`yq` 사용)

처음에는 `kustomize edit set image` 명령어를 사용하려 했으나, 이 명령어가 실행될 때 기존 YAML 파일의 포맷팅이 의도치 않게 변경되는 문제가 있었습니다.

저는 YAML 파일의 가독성은 정말 중요한 요소라고 생각해서, YAML 구조를 그대로 유지하면서 값만 수정할 수 있는 `yq` 도구를 선택했습니다.

```yaml
update-dev-manifest:
    needs: [ci-backend-dev, ci-frontend-dev]
    if: always() && (needs.ci-backend-dev.result == 'success' || needs.ci-frontend-dev.result == 'success')
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
        with:
          token: ${{ secrets.ACTION_TOKEN }}
          fetch-depth: 0

      # yq를 사용하여 backend 이미지의 newTag 값을 현재 커밋 해시로 교체
      - name: Update Backend Image Tag
        if: needs.ci-backend-dev.result == 'success'
        uses: mikefarah/yq@master
        with:
          cmd: yq -i '(.images[] | select(.name == "backend").newTag) = "${{ github.sha }}-dev"' kubernetes/overlays/dev/kustomization.yaml

      # frontend도 동일한 방식으로 처리
      - name: Update Frontend Image Tag
        if: needs.ci-frontend-dev.result == 'success'
        uses: mikefarah/yq@master
        with:
          cmd: yq -i '(.images[] | select(.name == "frontend").newTag) = "${{ github.sha }}-dev"' kubernetes/overlays/dev/kustomization.yaml
```

### 3-3. 무한 루프 방지와 Git 설정

마지막으로 변경된 kustomization.yaml을 저장소에 푸시해야 합니다.
여기서 주의할 점은, GitHub Actions가 푸시한 커밋이 다시 GitHub Actions를 트리거하여 무한 루프에 빠질 수 있다는 점입니다.

이를 방지하기 위해 커밋 메시지에 `[skip ci]`를 포함시켜, 이 커밋은 CI 파이프라인을 타지 않도록 설정했습니다.
또한, 작업 도중 다른 개발자가 커밋을 했을 경우를 대비해 `pull --rebase`를 수행하여 충돌을 예방했습니다.

```yaml
- name: Commit and push changes
  run: |
    git config --global user.name 'github-actions[bot]'
    git config --global user.email 'github-actions[bot]@users.noreply.github.com'
    
    git add kubernetes/overlays/dev/kustomization.yaml

    # 변경사항이 없으면 조용히 종료
    if git diff-index --quiet HEAD; then
      echo "No changes to commit"
      exit 0
    fi

    # [skip ci]: 이 커밋으로 인해 CI가 다시 도는 것을 방지
    git commit -m "ci: Update dev image tags to ${{ github.sha }} [skip ci]"
    
    # 원격 저장소의 최신 변경사항을 가져와 리베이스
    git pull --rebase origin ${{ github.ref_name }}
    git push origin ${{ github.ref_name }}
```

## 4. 마무리
---

이렇게 구축한 파이프라인 덕분에 이제 개발 브랜치에 코드를 푸시하기만 하면, 자동으로 이미지가 빌드되고 Kubernetes Manifest까지 업데이트됩니다.
더 이상 배포할 때마다 수동으로 태그를 수정할 필요가 없어졌고, kubectl apply를 실행하면 즉시 새로운 버전의 파드가 생성되는 것을 확인할 수 있습니다.

![](/assets/2026-01-31-kustomize-image-tag/action-result.png)
_GitHub Actions 실행 결과_

![](/assets/2026-01-31-kustomize-image-tag/action-result-push.png)
_자동으로 커밋된 kustomization.yaml 변경 사항_

간단한 자동화지만, 이를 통해 개발자의 실수를 줄이고 배포 프로세스의 복잡도를 낮출 수 있었습니다.
사실 이 Manifest를 수정하고 커밋하는 방법을 결정한 또 다른 이유는 이후 ArgoCD와 같은 GitOps 도구를 도입할 계획이 있기 때문인데요,
이 부분에 대해서는 저도 더 공부해보고 다음 기회에 기록해보겠습니다.

## 참고 자료
---

- [Kustomize 공식 문서 - Images](https://github.com/openshift/kubernetes-kubectl/blob/master/docs/book/pages/reference/kustomize.md#images)
- [GitHub Actions - dorny/paths-filter](https://github.com/dorny/paths-filter)
