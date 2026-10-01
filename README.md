# EPUB 리더 GitOps

`epub-anki-web-app`의 Kubernetes 배포 설정을 관리하는 독립 Git 저장소입니다.
운영 환경 하나를 사용하며 앱 코드는 인접한 소스 저장소에서 관리합니다.

## 파일 구성

- `kustomization.yaml`: 적용할 리소스 목록, namespace, 이미지 주소와 버전
- `deployment.yaml`: 단일 Pod, Recreate 배포, 헬스체크, 실행 권한과 리소스 설정
- `service.yaml`: 내부 접속용 ClusterIP Service
- `pvc.yaml`: SQLite와 EPUB 파일을 보관할 5Gi PVC
- `namespace.yaml`: 앱 namespace

## 이미지 버전 관리

현재 이미지 주소와 버전은 자리표시자입니다. 첫 배포 전에
`kustomization.yaml`의 `newName`을 실제 GHCR 주소로 바꾸고 `newTag`를 발행한
이미지 태그로 지정하세요. 운영에서는 `newTag` 대신 실제 `digest`를 고정합니다.

```yaml
images:
  - name: epub-reader
    newName: ghcr.io/YOUR_ACCOUNT/epub-reader
    digest: sha256:실제_이미지_digest
```

`deployment.yaml`의 `image: epub-reader`를 Kustomize가 위 주소와 버전으로
치환합니다. 이후 일반적인 코드 배포에서 CI가 변경할 파일은
`kustomization.yaml`의 `digest`입니다. 앱 CI가 이미지를 GHCR에 올린 다음 이
저장소를 갱신하도록 연결해야 하며, 현재 CI는 아직 구성하지 않았습니다.

## 배포 전 확인과 수동 적용

이 저장소 루트에서 실행합니다. 설정 생성은 클러스터에 변경을 적용하지 않습니다.

```sh
kubectl kustomize .
```

이 저장소는 운영 환경 하나를 위한 Kustomize 설정입니다. 단일 Pod와 `Recreate`
전략을 사용하므로 배포 중 짧은 중단이 발생합니다. `/data`에는 5Gi PVC를 연결합니다.
기본 StorageClass가 SQLite WAL에 적합한 블록 스토리지인지 먼저 확인하고,
필요하면 `pvc.yaml`에 `storageClassName`을 지정하세요. NFS는 사용하지 않습니다.
CPU/메모리 값은 초기값이며 최대 50MB EPUB 업로드를 측정해 조정해야 합니다.

비공개 GHCR 이미지는 namespace에 pull 인증 Secret을 준비하고 Deployment의
`spec.template.spec.imagePullSecrets`에 연결해야 합니다. 인증 값은 Git에 넣지 마세요.

```sh
kubectl apply -k .
kubectl -n epub-reader rollout status deployment/epub-reader
kubectl -n epub-reader port-forward service/epub-reader 3000:80
```

Service는 ClusterIP이므로 우선 port-forward로 접속합니다. 도메인, TLS, VPN 또는
인증 프록시는 실제 클러스터 구성에 맞춰 추가합니다. 앱에는 로그인 기능이 없습니다.
Ingress를 추가한다면 50MB 파일의 multipart 여유를 포함한 요청 크기와 타임아웃을
설정하세요. startup/liveness는 `/`, readiness는 SQLite에 접근하는 `/api/books`를
검사합니다. 개인 책장에 접근하지 않는 별도 볼륨으로 업로드, 위치 저장, 재시작 후
복원을 확인한 뒤 기존 데이터를 이전하세요.

## GitHub와 Argo CD 연결

로컬 Git 저장소만 초기화된 상태입니다. GitHub에 배포 저장소를 생성한 뒤 원격을
연결하고 첫 커밋을 push하세요. 커밋 메시지는 한글로 작성합니다.

Argo CD Application에는 다음 값을 지정합니다.

- 저장소: 새 GitOps 저장소의 Git URL
- 브랜치: `main`
- 경로: `.` (루트에 `kustomization.yaml`이 있습니다)
- 대상 namespace: `epub-reader`

Argo CD가 이 저장소의 Kustomize 설정을 동기화하도록 연결합니다. 기존 앱 저장소의
`deploy/` 경로는 더 이상 사용하지 않습니다. 첫 동기화 전에 이미지, 스토리지와
인증을 설정하고 자동 동기화 여부를 정하세요. Argo CD 설치와 Application 등록은
아직 수행하지 않았습니다.

## 데이터 보호와 복구

Namespace와 PVC에는 Argo CD의 자동 prune 및 Application
삭제 시 삭제를 막는 annotation을 넣었습니다. `kubectl delete`로 직접 삭제하는 것은
막지 않으므로 데이터가 있는 namespace/PVC를 삭제하지 마세요. PV reclaim policy와
데이터 백업/복원도 별도로 준비해야 합니다. DB 마이그레이션은 이미지 롤백으로
되돌아가지 않습니다.
