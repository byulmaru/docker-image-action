# docker-image-action

ARC runner에서 Docker daemon이나 DinD 없이 Docker image를 빌드하고 push하기 위한 composite action입니다.

이 액션은 `docker/setup-buildx-action`의 Kubernetes driver를 사용합니다. Workflow 실행 시 Buildx가 필요한 BuildKit 리소스를 Kubernetes 안에 on-demand로 생성하고, 빌드가 끝나면 해당 builder 수명 주기에 따라 정리됩니다.

## 방향

이 저장소의 액션은 다음 방향을 기준으로 합니다.

- standalone remote `buildkitd` Deployment를 상시 운영하지 않습니다.
- BuildKit은 workflow 실행 시 Buildx Kubernetes driver가 생성합니다.
- runner Pod 안에서 Docker daemon 또는 DinD를 사용하지 않습니다.
- BuildKit Pod는 기본적으로 `arc-runners` namespace에 생성됩니다.
- runner Pod는 `buildx-runner` ServiceAccount를 사용해야 합니다.
- 생성되는 BuildKit Pod는 기본적으로 `buildkit` ServiceAccount를 사용합니다.
- `buildkit` ServiceAccount는 Kubernetes API token mount를 비활성화하는 것을 전제로 합니다.

## 사용 예시

```yaml
jobs:
  image:
    runs-on: byulmaru-arc
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: byulmaru/docker-image-action@v1
        with:
          image: ghcr.io/byulmaru/my-app
          tags: |
            type=raw,value=latest
            type=sha
```

`tags`는 `docker/metadata-action`의 tag rule 문법을 그대로 사용합니다.

```yaml
with:
  image: ghcr.io/byulmaru/my-app
  tags: |
    type=raw,value=latest
    type=sha
    type=raw,value=release-${{ github.run_number }}
```

위 예시는 `image` 입력값을 기준으로 다음 image tag를 생성합니다.

```text
ghcr.io/byulmaru/my-app:latest
ghcr.io/byulmaru/my-app:sha-<short-sha>
ghcr.io/byulmaru/my-app:release-<run-number>
```

## 기본값

현재 기본값은 ARC 환경에서 Buildx Kubernetes driver를 바로 쓰기 위한 값으로 중앙화되어 있습니다.

```yaml
platforms: linux/arm64
node-selector: oci.oraclecloud.com/oke-is-preemptible=true
tolerations: key=oci.oraclecloud.com/oke-is-preemptible,operator=Exists,effect=NoSchedule
requests-cpu: 800m
requests-memory: 2Gi
limits-cpu: 1
limits-memory: 2Gi
```

`node-selector`는 기본적으로 OKE preemptible node를 선택합니다. 다른 node pool에서 빌드해야 하면 workflow에서 명시적으로 덮어씁니다.

```yaml
- uses: byulmaru/docker-image-action@v1
  with:
    image: ghcr.io/byulmaru/my-app
    node-selector: pool=build
```

## Kubernetes 권한

Workflow를 실행하는 runner Pod의 ServiceAccount는 Buildx가 `arc-runners` namespace에 BuildKit 리소스를 만들 수 있는 권한이 필요합니다.

권장 구성은 다음과 같습니다.

- runner Pod ServiceAccount: `buildx-runner`
- BuildKit Pod ServiceAccount: `buildkit`
- BuildKit namespace: `arc-runners`
- `buildkit` ServiceAccount: `automountServiceAccountToken: false`

`buildx-runner`에는 Buildx Kubernetes driver가 builder를 만들고 삭제하는 데 필요한 Kubernetes API 권한을 부여합니다. 반대로 실제 빌드를 수행하는 `buildkit` ServiceAccount에는 Kubernetes API token을 mount하지 않아 빌드 컨테이너가 Kubernetes API 권한을 갖지 않도록 합니다.

## Buildx Kubernetes Driver 옵션

Buildx Kubernetes driver에서 action 입력값으로 직접 제어하는 항목은 다음과 같습니다.

- `namespace`
- `replicas`
- `serviceaccount`
- `nodeselector`
- `tolerations`
- `requests.cpu`
- `requests.memory`
- `limits.cpu`
- `limits.memory`

`namespace`, `replicas`, `serviceaccount`는 workflow 입력값으로 노출하지 않고 액션 내부에서 고정합니다.

```yaml
namespace: arc-runners
replicas: 1
serviceaccount: buildkit
```

현재 composite action은 `node-selector`, `tolerations`, resource request/limit 기본값을 중앙화합니다. Workflow마다 반복해서 같은 BuildKit Pod 설정을 적지 않기 위한 목적입니다.

Buildx Kubernetes driver의 driver option만으로는 다음 항목을 직접 설정할 수 없습니다.

- `priorityClassName`
- preferred affinity

이 항목들이 필요해지면 action 입력값만으로 해결하지 않고, Kubernetes 쪽 스케줄링 정책이나 별도 구성 방식을 다시 검토해야 합니다.

현재는 preferred affinity를 포기하고 `node-selector: oci.oraclecloud.com/oke-is-preemptible=true`로 preemptible node를 직접 선택합니다. 따라서 해당 label을 가진 node가 없거나 taint/toleration이 맞지 않으면 BuildKit Pod가 스케줄링되지 않습니다.

`tolerations` 값에는 comma가 포함되므로 action 내부에서 Buildx driver option을 quote해서 전달합니다. quote가 없으면 Buildx가 `effect=NoSchedule`을 별도 driver option으로 해석해 `invalid driver option effect for driver kubernetes` 오류가 납니다.

## Inputs

| Input             | Default                                       | Description                                                                                         |
| ----------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `image`           | required                                      | tag를 제외한 image 이름입니다. 예: `ghcr.io/byulmaru/app`                                           |
| `tags`            | `type=sha`                                    | 줄바꿈으로 구분한 `docker/metadata-action` tag rule입니다.                                          |
| `context`         | `.`                                           | Docker build context 경로입니다.                                                                    |
| `file`            | `Dockerfile`                                  | Dockerfile 경로입니다.                                                                              |
| `push`            | `true`                                        | 빌드한 image를 registry에 push할지 여부입니다.                                                      |
| `platforms`       | `linux/arm64`                                 | 빌드 대상 platform입니다. 예: `linux/amd64,linux/arm64`                                             |
| `registry`        | `ghcr.io`                                     | `docker/login-action`에 사용할 registry host입니다. 빈 값이면 login을 생략합니다.                   |
| `username`        | `github.actor`                                | Registry username입니다.                                                                            |
| `password`        | `github.token`                                | Registry password 또는 token입니다.                                                                 |
| `requests-cpu`    | `800m`                                        | BuildKit Pod CPU request입니다.                                                                     |
| `requests-memory` | `2Gi`                                         | BuildKit Pod memory request입니다.                                                                  |
| `limits-cpu`      | `1`                                           | BuildKit Pod CPU limit입니다.                                                                       |
| `limits-memory`   | `2Gi`                                         | BuildKit Pod memory limit입니다.                                                                    |
| `node-selector`   | `oci.oraclecloud.com/oke-is-preemptible=true` | Buildx Kubernetes `nodeselector` option으로 전달됩니다. 기본값은 OKE preemptible node를 선택합니다. |
| `tolerations`     | OCI preemptible Exists toleration             | Buildx Kubernetes `tolerations` option으로 전달됩니다.                                              |
| `build-args`      | empty                                         | 줄바꿈으로 구분한 build argument입니다.                                                             |
| `labels`          | empty                                         | 줄바꿈으로 구분한 `docker/metadata-action` labels 입력값입니다.                                     |
| `target`          | empty                                         | 선택 사항입니다. Dockerfile build target입니다.                                                     |

## Outputs

| Output     | Description                                        |
| ---------- | -------------------------------------------------- |
| `imageid`  | 빌드된 image ID입니다.                             |
| `digest`   | 빌드된 image digest입니다.                         |
| `metadata` | `docker/build-push-action`의 build metadata입니다. |

## 배포 전 체크리스트

- `docker-image-action` repo를 GitHub에 생성합니다.
- 이 저장소를 remote에 push합니다.
- `byulmaru/docker-image-action@v1` 태그를 생성합니다.
- infrastructure repo의 ARC/ServiceAccount/RBAC 변경사항을 commit하고 apply합니다.
- 실제 workflow에서 `byulmaru/docker-image-action@v1`로 image build와 push를 검증합니다.
