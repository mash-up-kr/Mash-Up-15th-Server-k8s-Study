# 5장 볼륨·마운트·클레임으로 **데이터 퍼시스턴시** 보장하기

> 파드가 죽어도 데이터는 살아‑있어야 함. 그래서 **스토리지 계층**을 쿠버네티스가 추상화했음.

---

## 1. 컨테이너‑파일시스템 구조  
- **컨테이너 이미지 레이어** + **기록 가능한 레이어**로 구성됨  
- 기록 가능 레이어는 컨테이너 생애주기를 따르므로 **파드 재시작 시 데이터 유실** 발생

---

## 2. 파드 수준 임시 스토리지

| 유형 | 라이프사이클 | 한계/특징 | 흔한 용도 |
|------|-------------|-----------|-----------|
| **EmptyDir** | 파드와 동일 | 파드 교체 시 초기화 | 임시 캐시, 세션 파일 |
| **HostPath** | 노드 디스크 | 파드가 **항상 같은 노드**에 떠야 함 · 보안 위험(디렉터리 제한 없음) | DB 테스트, 단노드 캐시 |


> **정리**: 임시 저장은 쉽지만 복제·자가수복성 떨어짐 → **클러스터 전역 스토리지** 필요함.

---

## 3. 클러스터 전역 스토리지 추상화

### 3‑1. 핵심 리소스
- **PersistentVolume(PV)**  
  - 실제 스토리지 조각을 **관리자/클러스터**가 미리 등록한 객체  
  - NFS·EBS·Ceph·CSI 등 **스토리지 구현 세부사항 캡슐화**  
- **PersistentVolumeClaim(PVC)**  
  - 파드 또는 개발자가 “_이만큼 주세요_” 하고 **요청**하는 객체  
  - 일대일 바인딩 후 파드가 `volumeMounts` 로 사용

### 3‑2. 프로비저닝 방식  
| 방식 | 절차 | 장단점 |
|------|------|--------|
| **정적** | ① 관리자가 PV 작성 → ② 사용자가 PVC 작성 후 수동 매칭 | 예측 가능하지만 운영 부담 |
| **동적** | ① 사용자가 PVC만 작성 → ② `StorageClass`가 조건 맞는 PV **자동 생성** | 운영 편리 · 스토리지 낭비 최소화 |

### 3‑3. 핵심 속성  
- **AccessModes**: `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`  
- **ReclaimPolicy**: `Retain`, `Recycle`(deprecated), `Delete`  
- **StorageClass**: 동적 프로비저닝·QoS·압축·암호화 등 **스토리지 기능 템플릿**

> PV/PVC 분리 덕분에 “애플리케이션(개발) ≠ 스토리지(운영)” 역할 구분이 명확해짐 :contentReference[oaicite:3]{index=3}

---

## 4. 빠르게 써먹는 YAML 샘플

```yaml
# StorageClass 예시 (동적‑프로비저닝용)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
---

# PVC 예시 – 5Gi SSD 요청
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: [ ReadWriteOnce ]
  storageClassName: gp3-ssd
  resources:
    requests:
      storage: 5Gi
---

# 파드에서 사용
volumeMounts:
  - mountPath: /var/lib/data
    name: data
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
