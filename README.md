# Hyper-V 기반 Kubernetes 멀티 노드 클러스터 구축 자동화

본 프로젝트는 **Windows 11** 환경의 **Hyper-V**에서 **PXE 서버와 Ansible**을 활용하여 **Kubernetes 멀티 노드 클러스터**를 자동으로 구축하는 프로젝트입니다. 최소한의 수작업만으로 OS 설치부터 Kubernetes 클러스터 구성까지 무인화하여, 빠르게 멀티 노드 환경을 구축할 수 있도록 구성했습니다.

---

## 📌 목차 (Table of Contents)

1. [프로젝트 개요](#프로젝트-개요)
2. [아키텍처 구성](#아키텍처-구성)
3. [요구 사항 (Requirements)](#요구-사항-requirements)
4. [구축 방법 (Installation)](#구축-방법-installation)
5. [사용 예시 (Usage)](#사용-예시-usage)
6. [결과 검증 및 테스트](#결과-검증-및-테스트)
7. [트러블 슈팅 경험](#트러블-슈팅-경험)
8. [향후 개선 방향](#피드백-및-향후-디벨롭-방향성)
9. [참고 자료](#참고-자료)

---

## 프로젝트 개요

### 목표
- Hyper-V 기반 Kubernetes 클러스터 (컨트롤러, 스토리지, 워커 노드) 자동화 구축
- PXE 서버로 Rocky Linux 9 OS를 자동 설치
- Ansible로 Kubernetes 환경 및 CNI 구성을 무인 자동화

### 프로젝트 범위
- Hyper-V 환경 설정 (VM 생성 및 가상 스위치 구성)
- PXE 서버 및 OS 자동 설치 (Rocky Linux 9)
- Ansible을 통한 Kubernetes 자동 설치 및 CNI 설정
- Kubernetes 클러스터 검증 및 간단한 워크로드 배포 테스트

---

## 아키텍처 구성

### 물리 아키텍처

본 프로젝트에서는 Windows 11 환경 위에 Hyper-V 기반의 멀티 노드 Kubernetes 클러스터를 구축합니다.

- 2대의 Windows 11 데스크탑(A, B) 위에서 Hyper-V 활성화 후 가상 스위치 생성.
- External Virtual Switch를 통해 물리 랜 카드와 브리지로 연결하여 외부 네트워크(인터넷) 접근 가능하도록 구성.
- PXE 및 Kubernetes 네트워크 구성을 위해 VM 당 3개의 NIC 사용:
  - **NIC1:** Management용 (PXE 서버 및 Ansible 관리 용도)
  - **NIC2 (VLAN 5):** Internal 네트워크 용도 (내부 Kubernetes 트래픽)
  - **NIC3 (VLAN 6):** External 네트워크 용도 (외부 접근 용도, 인터넷 등)

### VM NIC 구성 및 VLAN

| 용도 | NIC 구성 | VLAN |
|------|----------|------|
| Management (PXE/Ansible) | NIC1 | 미설정 |
| Internal Network (K8s 통신) | NIC2 | VLAN 5 |
| External Network (외부) | NIC3 | VLAN 6 |

### VM IP 및 MAC

| 노드 | 호스트명 | Management NIC1 | Internal NIC2 | External NIC3 | MAC |
|------|------------------------|------------------|-----------------|-----------------|---------------------|
| Bootstrap | bootstrap.example.com | 192.168.0.101 | 172.16.0.101 | 172.26.0.101 | 02:AA:00:00:01:01 |
| Controller | controller1.example.com | 192.168.0.110 | 172.16.0.110 | 172.26.0.110 | 02:AA:00:00:01:10 |
| Storage | storage1.example.com | 192.168.0.200 | 172.16.0.200 | 172.26.0.200 | 02:AA:00:00:02:00 |
| Compute1 | compute1.example.com | 192.168.0.120 | 172.16.0.120 | 172.26.0.120 | 02:BB:00:00:01:20 |
| Compute2 | compute2.example.com | 192.168.0.121 | 172.16.0.121 | 172.26.0.121 | 02:BB:00:00:01:21 |
| Compute3 | compute3.example.com | 192.168.0.122 | 172.16.0.122 | 172.26.0.122 | 02:BB:00:00:01:22 |

---

## 요구 사항 (Requirements)

- Windows 11 (Hyper-V 활성화 필수)
- Rocky Linux 9 ISO
- Ansible 2.9 이상 (권장 2.13 이상)
- Kubernetes v1.30
- Container Runtime: containerd 또는 CRI-O
- CNI 플러그인: Flannel 또는 Calico

---

## 구축 방법 (Installation)

### PXE 서버 구성
- 부트스트랩 노드 VM 생성 (Rocky Linux 9 설치)
- DHCP/TFTP 서버 설정 (`/etc/dhcp/dhcpd.conf`)
- TFTP 디렉터리 설정 (`/var/lib/tftpboot`)
- Kickstart 설정으로 OS 무인 설치 자동화

### Ansible 환경 구축
- Ansible 설치 및 인벤토리 구성
- SSH 키 배포 (비밀번호 없는 접속 환경 구축)
- Kubernetes 설치 및 설정 플레이북 실행

```bash
ansible-playbook -i inventory.ini k8s_install.yml
```

---

## 사용 예시 (Usage)

1. VM PXE 부팅 후 OS 자동 설치
2. Ansible 플레이북 실행하여 Kubernetes 설치 자동화
3. Kubernetes 노드 상태 확인

```bash
kubectl get nodes
```

---

## 결과 검증 및 테스트

- Kubernetes 클러스터 정상 동작 확인

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc
```

---

## 트러블 슈팅 경험

- **DHCP IP 할당 문제**: MAC 주소 기반 IP 예약 설정을 통해 해결
- **Hyper-V VM 세대 문제**: BIOS 기반 PXE 부팅을 위해 1세대(Gen 1) VM 사용
- **네트워크 어댑터 인식 문제**: 레거시 네트워크 어댑터 추가하여 해결

---

## 피드백 및 향후 디벨롭 방향성

- PXE 서버 자동화로 OS 설치 및 고정 IP 부여
- Kubernetes 클러스터 구축 자동화로 반복 작업 감소
- 향후 AWS Collection을 사용한 클라우드 확장 고려

---

## 참고 자료

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Ansible Documentation](https://docs.ansible.com/)
- [PXE Booting Example Blog](https://pinetreeday.tistory.com/176)
- [Rocky Linux](https://rockylinux.org)

---

※ 본 문서는 프로젝트 발표 및 문서화를 위해 제작되었습니다. 세부 설정 파일 및 스크립트는 각 폴더를 참조해 주세요.
