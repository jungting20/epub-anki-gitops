# 프로젝트 지침

- 커밋 메시지는 반드시 한글로 작성한다.
- 이 저장소는 EPUB 리더의 Kubernetes 배포 설정을 관리한다.
- 일반적인 이미지 배포는 `kustomization.yaml`의 이미지 digest를 갱신한다.
- YAML 변경 후 `kubectl kustomize .`로 최종 설정을 확인한다.
- 인증 정보와 실제 사용자 데이터를 Git에 저장하지 않는다.
