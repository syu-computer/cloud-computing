# Libvirt 실습 가이드

본 문서는 클라우드컴퓨팅 수업의 libvirt 튜토리얼에 대한 실습 가이드임.

## 1. 실습 준비
* 실습 시에 원활한 작업을 위해 슈퍼유저 권한을 가지고 로그인

### 1-1. 실습을 위한 권한 설정 (슈퍼유저로 로그인)
```bash
sudo -i
```
* `pwd` 명령어를 입력해보면 root 계정의 home 경로인 `/root`로 변경된 것을 확인할 수 있음

### 1-2. 클라우드 전용 Ubuntu OS 이미지 파일 다운로드
```bash
wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
```

### 1-3. 현재 경로에서 다운로드된 img 파일 확인
```bash
ls -l
```

### 1-4. 실습에 필요한 라이브러리 일괄 설치
```bash
apt update && apt install -y libvirt-clients libvirt-daemon-system virtinst cloud-image-utils python3-libvirt guestfs-tools
```

## 2. 가상 네트워크 정의
간단한 가상 네트워크 정의 xml 파일을 vir-network.xml라는 이름으로 작성하기

### 2-1. `vi` 텍스트 에디터를 사용하여 `vir-network.xml` 파일을 생성

```bash
vi vir-network.xml
```

### 2-2. `vi` 에디터가 열리면 입력 모드 `i` 입력 후, 아래의 XML 코드를 입력 및 저장하고 나오기 (`esc 키` -> :`wq`)
```xml
<network>
  <name>vir-network</name>
  <bridge name="virbr1"/>
  <forward mode="nat"/>
  <ip address="192.168.123.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.123.2" end="192.168.123.254"/>
    </dhcp>
  </ip>
</network>
```

### 2-3. 작성된 문서 확인
```bash
cat vir-network.xml
```

### 2-4. 가상 네트워크 정의
```bash
virsh net-define vir-network.xml
```

### 2-5. 가상 네트워크 목록에서 정의된(inactive 상태) 네트워크 확인
```bash
virsh net-list --all
```

### 2-6. 현재 리눅스 네트워크 구성 확인
```bash
ip address
```
* 아직 `virbr1` 브릿지가 보이지 않음

### 2-7. 가상 네트워크 시작
```bash
virsh net-start vir-network
```

### 2-8. 가상 네트워크 목록에서 시작된(active 상태) 네트워크 확인
```bash
virsh net-list --all
```

### 2-9. 현재 리눅스 네트워크 구성 확인
```bash
ip address
```
* `virbr1` 브릿지가 확인됨

### 2-10. 가상 네트워크 가 부팅 시 자동으로 시작되도록 설정
```bash
virsh net-autostart vir-network
```

### 2-11. 가상 네트워크 정보 조회
```bash
virsh net-info vir-network
```

## 3. 가상 머신의 초기화 작업 준비
* cloud-init: 가상 머신 또는 클라우드 인스턴스에서 초기화 및 설정 작업을 자동화하기 위한 프로그램

### 3-1. `vi` 텍스트 에디터를 사용하여 `user-data` 파일을 생성

```bash
vi user-data
```

### 3-2. `vi` 에디터가 열리면 입력 모드 `i` 입력 후, 아래의 YAML 코드를 입력 및 저장하고 나오기 (`esc 키` -> :`wq`)
```yaml
#cloud-config
hostname: vm01
manage_etc_hosts: true
users:
  - name: ubuntu
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups:
      - users
      - admin
    home: /home/ubuntu
    shell: /bin/bash
    lock_passwd: false
ssh_pwauth: true
disable_root: false
chpasswd:
  list:
    - ubuntu:1111
  expire: false

```
* `#cloud-config`는 클라우드 컴퓨팅 환경에서 가상 머신(VM) 또는 서버 인스턴스를 초기 설정할 때 사용되는 특수한 주석임
* `Cloud-init`이 이를 인식하고 실행

### 3-3. `vm1`을 위해 작성된 `user-data` 확인
```bash
cat user-data
```

### 3-4. `vm2`을 위해 작성된 `vm1`의 `user-data`를 복사
```bash
cp user-data user-data2
```

### 3-5. `vi` 텍스트 에디터를 사용하여 `user-data2` 파일을 수정

```bash
vi user-data2
```

### 3-6. `vi` 에디터가 열리면 입력 모드 `i` 입력 후, 아래 YAML 코드 수정 및 저장하고 나오기 (`esc 키` -> :`wq`)
```yaml
#cloud-config
hostname: vm02
manage_etc_hosts: true
users:
  - name: ubuntu
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups:
      - users
      - admin
    home: /home/ubuntu
    shell: /bin/bash
    lock_passwd: false
ssh_pwauth: true
disable_root: false
chpasswd:
  list:
    - ubuntu:1111
  expire: false

```
* `hostname`의 `vm01`을 `vm02`로 변경

### 3-7. 작성된 문서 확인
```bash
cat user-data2
```

### 3-8. `vm1`과 `vm2`의 user-data를 각각 포함하는 qcow2 이미지들을 생성
```bash
cloud-localds -v -d qcow2 vm01-base.qcow2 user-data
```
```bash
cloud-localds -v -d qcow2 vm02-base.qcow2 user-data2
```
* cloud-localds: 클라우드 인스턴스를 시작할 때 사용할 초기화 파일을 생성하는 명령어
* -v: 명령 실행 중 상세한 출력을 표시
* -d qcow2: 생성할 디스크 이미지 파일의 포맷을 지정하는 옵션
* vm01-base.qcow2: 생성할 디스크 이미지 파일 이름
* user-data: 클라우드 인스턴스에 필요한 초기화 파일을 생성하는 데 사용되는 파일 이름

### 3-9. 이미지 파일들의 위치 조정
```bash
cp vm01-base.qcow2 /var/lib/libvirt/images/
cp vm02-base.qcow2 /var/lib/libvirt/images/
cp focal-server-cloudimg-amd64.img /var/lib/libvirt/images/focal-server-cloudimg-amd64-vm01.img
cp focal-server-cloudimg-amd64.img /var/lib/libvirt/images/focal-server-cloudimg-amd64-vm02.img

```
* /var/lib/libvirt/images: KVM 또는 QEMU 가상화 플랫폼에서 가상 디스크 이미지를 저장하는 기본 디렉토리

### 3-10. 복사한 이미지 파일들 확인
```bash
ls /var/lib/libvirt/images/
```
* 총 4개의 파일이 확인되어야 함.


## 4. 가상 머신 정의 및 시작
* 가상 머신을 정의하고 시작하는 두 가지 방법이 있음
1) `virt-install`
2) `virsh define` & `virsh start `

### 4-1. `virt-install`를 통한 가상 머신 정의 및 시작
```bash
virt-install \
--name=vm1 \
--ram=2048 \
--vcpus=2 \
--os-variant=ubuntu20.04 \
--disk path=/var/lib/libvirt/images/focal-server-cloudimg-amd64-vm01.img,device=disk,bus=virtio \
--disk path=/var/lib/libvirt/images/vm01-base.qcow2,device=disk,bus=virtio \
--network network=vir-network,model=virtio,mac=52:54:00:12:34:58 \
--noautoconsole \
--console pty \
--serial pty \
--boot hd \
--import

```

### 4-2. 가상 머신 목록에서 `running` 상태인 `vm1` 가상 머신 확인
```bash
virsh list --all
```

### 4-3. 가상 머신 접속
```bash
virsh console vm1
```

### 4-4. 가상 머신 접속 후 계정/비번 입력하여 로그인
* login ID: ubuntu
* login PW: 1111

### 4-5. 가상 머신에서 빠져나오기
* Escape character is `Ctrl + ]`

### 4-6. `vi` 텍스트 에디터를 사용하여 `vm2.xml` 파일을 생성
```bash
vi vm2.xml
```
* `vm2`의 정의를 위함

### 4-7. `vi` 에디터가 열리면 입력 모드 `i` 입력 후, 아래 XML 코드와 같이 입력 및 저장하고 나오기 (`esc 키` -> :`wq`)
```xml
<domain type='kvm'>
  <name>vm2</name>
  <memory unit='KiB'>2097152</memory>
  <vcpu placement='static'>2</vcpu>
  <os>
    <type arch='x86_64' machine='pc-q35-2.9'>hvm</type>
    <boot dev='hd'/>
  </os>
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/focal-server-cloudimg-amd64-vm02.img'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/vm02-base.qcow2'/>
      <target dev='vdb' bus='virtio'/>
    </disk>
    <interface type='network'>
      <mac address='52:54:00:12:34:59'/>
      <source network='vir-network'/>
      <model type='virtio'/>
    </interface>
    <console type='pty'>
        <target port='0'/>
    </console>
    <serial type='pty'>
        <target port='0'/>
    </serial>
  </devices>
</domain>

```
* 가상 머신 이름: `vm2`
* Ubuntu 디스크 이미지: `/var/lib/libvirt/images/focal-server-cloudimg-amd64-vm02.img`로
* user-data를 위한 디스크 이미지: `/var/lib/libvirt/images/vm02-base.qcow2`
* 인터페이스의 mac 주소(mac 주소가 서로 달라야 가상머신 간에 통신 가능): `52:54:00:12:34:59`

### 4-8. 작성된 문서 확인
```bash
cat vm2.xml
```

### 4-9. 가상 머신 정의
```bash
virsh define vm2.xml
```

### 4-10. 가상 머신 목록에서 정의된(shut off 상태) 가상 머신 확인
```bash
virsh list --all
```

### 4-11. 가상 머신 시작
```bash
virsh start vm2
```

### 4-12. 가상 머신 목록에서 정의된(running 상태) 가상 머신 확인
```bash
virsh list --all
```

### 4-13. 가상 머신들 기본정보 출력
```bash
virsh dominfo vm1 && virsh dominfo vm2
```
* `hvm(Hardware Virtual Machine)`: 가상 머신(VM)이 하드웨어 가상화 기술(`Intel VT-x` 또는 `AMD-V`)을 사용하여 구동된다는 것을 의미

### 4-14. 가상 머신들에 할당된 IP 주소를 확인
```bash
virsh domifaddr vm1 && virsh domifaddr vm2
```
* VM에 할당된 IP는 환경에 따라 다를 수 있음

### 4-15. 가상 머신 접속
```bash
virsh console vm2
```

### 4-16. 가상 머신 접속 후 계정/비번 입력하여 로그인
* login ID: ubuntu
* login PW: 1111

### 4-17. 가상 머신 간의 연결 상태를 테스트
```bash
ping <vm-ip-address>
```
* `<vm-ip-address>`에 상대방 VM의 인터페이스에 할당되어 있는 IP 주소를 기입하여 명령어 실행
* VM에 할당된 IP는 환경에 따라 다를 수 있음
* 아래는 실행 결과
```bash
ubuntu@vm02:~$ ping  192.168.123.78
PING 192.168.123.78 (192.168.123.78) 56(84) bytes of data.
64 bytes from 192.168.123.78: icmp_seq=1 ttl=64 time=3.59 ms
64 bytes from 192.168.123.78: icmp_seq=2 ttl=64 time=0.807 ms
64 bytes from 192.168.123.78: icmp_seq=3 ttl=64 time=0.902 ms
64 bytes from 192.168.123.78: icmp_seq=4 ttl=64 time=0.589 ms
64 bytes from 192.168.123.78: icmp_seq=5 ttl=64 time=0.888 ms
64 bytes from 192.168.123.78: icmp_seq=6 ttl=64 time=1.06 ms
```

### 4-18. 가상 머신에서 빠져나오기
* Escape character is `Ctrl + ]`


## 5. 가상 머신의 블록(block)

### 5-1. 가상 머신의 블록 장치 목록을 나열
```bash
virsh domblklist vm2
```
* 아래와 같이 두개의 블록 디스크 확인 가능
```bash
 Target   Source
------------------------------------------------------------------------
 vda      /var/lib/libvirt/images/focal-server-cloudimg-amd64-vm02.img
 vdb      /var/lib/libvirt/images/vm02-base.qcow2
```

### 5-2. 가상 머신의 블록 장치에 대한 정보를 조회
```bash
virsh domblkinfo vm2 vda
```
* Capacity:       `vda` 디스크의 전체 용량임 (디스크가 저장할 수 있는 최대 데이터 양을 바이트 단위로 나타냄)
* Allocation:     현재 `vda` 디스크에 실제로 할당된 공간의 양
* Physical:       `vda` 디스크에 실제로 사용되고 있는 물리적 공간의 양

### 5-3. 가상 머신에서 블록 장치의 I/O 통계 정보 조회
```bash
virsh domblkstat vm1 vda
```
* rd_req (Read Requests):  `vda` 디스크에 대한 총 읽기 요청의 수
* rd_bytes (Read Bytes): 총 읽기 요청을 통해 읽혀진 데이터의 양
* wr_req (Write Requests): `vda` 디스크에 대한 총 쓰기 요청의 수
* wr_bytes (Write Bytes): 총 쓰기 요청을 통해 쓰여진 데이터의 양
* flush_operations: `vda` 디스크에 대한 플러시(버퍼를 비움) 연산의 수
* rd_total_times (Total Time for Read Operations in nanoseconds): 읽기 요청의 총 처리 시간
* wr_total_times (Total Time for Write Operations in nanoseconds): 쓰기 요청의 총 처리 시간
* flush_total_times (Total Time for Flush Operations in nanoseconds): 플러시 연산의 총 처리 시간
* 요금과 관련된 항목이라 말할 수 있는 항목은 wr_bytes (Write Bytes)와 rd_bytes (Read Bytes) 


## 6. 가상 머신의 일시 중단(suspend) 및 재게(resume) 

### 6-1. 가상 머신의 일시 중단(suspend)
```bash
virsh suspend vm2
```

### 6-2. 가상 머신 목록에서 일시정지된(paused 상태) 가상 머신 확인
```bash
virsh list
```
* suspend한 vm의 상태는 paused 상태로 변경되어 있음

### 6-3. 가상 머신의 재게(resume) 
```bash
virsh resume vm2
```
* resume한 vm의 상태는 running 상태로 변경되어 있음

### 6-4. 가상 머신 목록에서 가상 머신 상태 확인
```bash
virsh list
```
* suspend한 vm의 상태는 running 상태로 변경되어 있음


## 7. 가상 머신의 종료(shutdown, destory)와 시작(start)

### 7-1. 가상 머신의 종료
* `destroy`: 전원을 갑자기 꺼버리는 것과 동일함
* `shutdown`: 가상 머신에게 정상적인 종료를 요청
```bash
virsh destroy vm2
```
  
### 7-2. 가상 머신 목록에서 종료된(shut off 상태) 가상 머신 확인
```bash
virsh list --all
```
* destroy한 vm의 상태는 shut off 상태로 변경되어 있음

### 7-3. 가상 머신의 시작
```bash
virsh start vm2
```

### 7-4. 가상 머신 목록에서 가상 머신 상태 확인
```bash
virsh list
```
* vm의 상태는 running 상태로 변경되어 있음


## 8. 가상 머신의 스냅샷(Snapshot)
* 가상 머신의 현재 상태를 캡처하는 기술
* 가상 머신의 메모리 상태, 디스크 이미지 및 가상 머신의 상태 정보를 포함
* 가상 머신의 이전 상태를 저장하므로 가상 머신에서 오류가 발생한 경우 이전 상태로 복원할 수 있음
* 스냅샷은 가상 머신의 특정 시점에서 생성할 수 있음

### 8-1. 가상 머신의 스냅샷을 커멘드 라인으로 생성
```bash
virsh snapshot-create-as vm2 --name initial-version
```

### 8-2. 가상 머신의 스냅샷 목록 확인
```bash
virsh snapshot-list vm2
```

### 8-3. 가상 머신 내부에 파일을 생성
```bash
virsh console vm2
```
* vm2 가상머신에 접속
```bash
touch hello_vm2
```
* vm 가상 머신 내에 빈 파일 생성
* 추후에 스냅샷으로 복원할 경우 파일이 사라짐을 확인하기 위함
```bash
ls -l
```
* 생성된 파일 확인
  
### 8-4. 가상 머신에서 빠져나오기
* Escape character is `Ctrl + ]`

### 8-5. 가상 머신 `vm2`의 스냅샷 `initial-version`으로 되돌리기(revert)
```bash
virsh snapshot-revert vm2 initial-version
```

### 8-6. 가상 머신 내부에 파일 확인
```bash
virsh console vm2
```
* vm2 가상머신에 접속
```bash
ls -l
```
* 기존 hello_vm2 파일이 없음을 확인 (`initial-version`의 스냅샷으로 되돌아갔다는 의미임)
  
### 8-7. 가상 머신에서 빠져나오기
* Escape character is `Ctrl + ]`

### 8-8. 가상 머신 스냅샷 삭제
```bash
virsh snapshot-delete vm2 initial-version
```

### 8-9. 가상 머신 스냅샷 목록 확인
```bash
virsh snapshot-list vm2
```


## 9. 가상 머신의 네트워크 구성 확인하기

### 9-1. 현재 호스트에서 사용 가능한 가상 네트워크 목록을 출력
```bash
virsh net-list --all
```

### 9-2. 시스템의 네트워크 인터페이스 목록 확인
```bash
virsh iface-list
```
* 가상 환경에서 네트워크 구성과 트러블슈팅을 할 때 유용

### 9-3. 가상 머신의 네트워크 인터페이스 목록 출력
```bash
virsh domifaddr vm2
```
* 가상 인터페이스의 MAC 주소(2계층)와 IP 주소(3계층) 확인 가능

### 9-4. 가상 머신의 특정 네트워크 인터페이스에 대한 네트워크 I/O 통계 정보 출력
* 가상 머신의 인터페이스 사용량과 관련된 세부 정보를 확인할 때 사용
* 빌링, 네트워크 트래픽 분석, 성능 모니터링, 문제 해결 등에 유용
```bash
virsh domifstat vm2 vnet3
```
* `rx_bytes`: 수신된 총 바이트 수
* `rx_packets`: 수신된 총 패킷 수
* `rx_errs`:수신 과정에서 발생한 에러의 수 (에러가 발생하면 패킷이 손상되었거나 완전히 전달되지 않았을 수 있음)
* `rx_drop`: 드롭된 수신 패킷 수 (인터페이스에서 버퍼 오버플로우 등의 이유로 처리하지 못하고 버린 수신 패킷의 수)
* `tx_bytes`: 송신된 총 바이트 수
* `tx_packets`: 송신된 총 패킷 수
* `tx_errs`:송신 과정에서 발생한 에러의 수 (에러가 발생하면 패킷이 손상되었거나 완전히 전달되지 않았을 수 있음)
* `tx_drop`: 드롭된 송신 패킷 수 (인터페이스에서 버퍼 오버플로우 등의 이유로 처리하지 못하고 버린 수신 패킷의 수)


## 10. 가상 머신에 간단한 웹서버 구동하기

### 10-1. 가상머신에 콘솔로 접속
```bash
virsh console vm2
```

### 10-2. Apache 웹 서버 설치
```bash
sudo apt update
```
* 패키지 매니저를 업데이트하여 최신 패키지 목록을 가져옴.
```bash
sudo apt install apache2 -y
```
* 아파치 웹 서버를 설치

### 10-3. 웹 페이지(index.html) 수정
```bash
sudo vi /var/www/html/index.html
```
* `/var/www/html/`: Linux 시스템에서 Apache 웹 서버의 기본 웹 컨텐츠 디렉토리
* `index.html`: 웹 디렉토리의 기본 문서
```bash
:%d
```
* 명령 모드에서 위 명령을 입력하여 모든 내용을 지움
```html
<!DOCTYPE html>
<html>
<head>
   <title>Hello, I am VM2</title>
</head>
<body>
   <h1>Hello, I am VM2</h1>
</body> 
</html>
```
* 입력 모드로 전환 후, 위 내용을 작성
* 명령 모드로 전환 후, 저장하고 나오기 (:`wq`)
```bash
curl 127.0.0.1
```
* 현재 구동되고 있는 웹서버에 웹페이지 요청
* `index.html`의 내용이 응답으로 나타나는지 확인

### 10-4. 윈도우에서 가상머신까지 접근할 수 있도록 네트워크 환경 구성
* 하이퍼바이저 역할을 수행하는 우분투에서 아래와 같은 명령어 필요
```bash
iptables -t nat -I PREROUTING -d [우분투의 IP] -p tcp --dport 8080 -j DNAT --to-destination [Apache 서버가 구동중인 가상머신의 IP]:80
```
* NAT (Network Address Translation) 테이블을 수정하겠다는 의미임
* 외부에서 우분투 서버의 특정 포트(8080)로 들어오는 TCP 트래픽을 가상 머신에서 구동 중인 아파치 서버의 80번 포트로 전달
```bash
iptables -t filter -I FORWARD -p tcp -d [Apache 서버가 구동중인 가상머신의 IP] --dport 80 -j ACCEPT
```
* NAT 규칙에 의해 변경된 목적지 주소 (가상 머신의 IP)로 가는 트래픽을 허용함의 의미

### 10-3. 윈도우의 웹브라우저에서 아파치 서버 접근하기
* 윈도우 웹브라우저에서 `[우분투의 IP]:8080` 주소로 접근


## 11. libvirt-python으로 가상 머신 생성

### 11-1. 기존 vm2을 destroy 및 undefine
```bash
virsh destroy vm2
```
```bash
virsh undefine vm2
```

### 11-2. 가상 머신 상태 확인
```bash
virsh list --all
```

### 11-3. `vi` 텍스트 에디터를 사용하여 `script.py` 파일을 생성
```bash
vi script.py
```

### 11-4. `vi` 에디터가 열리면 입력 모드 `i` 입력 후, 아래의 코드를 입력 및 저장하고 나오기 (`esc 키` -> :`wq`)
```python
import libvirt
import sys

if len(sys.argv) < 2:
    print("Usage: python3 vm_manager.py [create|list|suspend|resume|snapshot]")
    sys.exit(1)

action = sys.argv[1]

# 연결할 libvirt URI를 설정
username = 'test'
ip = '127.0.0.1'
uri = f'qemu+ssh://{username}@{ip}/system'
conn = libvirt.open(uri)

if conn is None:
    print("Failed to open connection to the hypervisor")
    sys.exit(1)

try:
    if action == "create":
        # 생성할 도메인의 XML 정의를 파일에서 불러오기
        with open("/root/vm2.xml", "r") as file:
            domain_xml = file.read()

        # 도메인을 정의하고 저장
        domain = conn.defineXML(domain_xml)
        if domain is None:
            print("Failed to define a domain.")
            sys.exit(1)

        # 도메인을 생성
        if domain.create() < 0:
            print("Cannot boot the domain.")
            sys.exit(1)
        print(f"Domain {domain.name()} created and started.")

    elif action == "list":
        # 모든 도메인을 조회
        vms = conn.listDomainsID()
        print("Current VMs:")
        for vm_id in vms:
            vm = conn.lookupByID(vm_id)
            print(f"- {vm.name()}")
            state, maxmem, mem, cpus, cput = vm.info()
            print('The state is ' + str(state))
            print('The max memory is ' + str(maxmem))
            print('The memory is ' + str(mem))
            print('The number of cpus is ' + str(cpus))
            print('The cpu time is ' + str(cput))

    elif action == "suspend":
        # 모든 도메인을 일시 정지
        vms = conn.listDomainsID()
        for vm_id in vms:
            vm = conn.lookupByID(vm_id)
            vm.suspend()
        print("All running domains have been suspended.")

    elif action == "resume":
        # 모든 도메인을 재개
        vms = conn.listDomainsID()
        for vm_id in vms:
            vm = conn.lookupByID(vm_id)
            if vm.info()[0] == 3:  # 3: Paused
                vm.resume()
        print("All paused domains have been resumed.")

    elif action == "snapshot":
        # 첫 번째 도메인에 대해 스냅샷 생성
        vm_id = conn.listDomainsID()[0]
        vm = conn.lookupByID(vm_id)
        snap = vm.snapshotCreateXML("""
        <domainsnapshot>
            <name>Snapshot1</name>
            <description>An example snapshot</description>
        </domainsnapshot>
        """)
        if snap is None:
            print("Failed to create snapshot.")
            sys.exit(1)
        print(f"Snapshot created for domain {vm.name()}.")

finally:
    # 연결 종료
    conn.close()

```

## 11-5. 파이썬 스크립트 실행을 통한 가상 머신 관리
* 파이썬 스크립트를 사용하여 다양한 가상 머신 관리 작업을 수행할 수 있음

* 가상 머신 생성
```bash
python3 script.py create
```

* 가상 머신 목록 조회
```bash
python3 script.py list
```

* 가상 머신 일시 정지
```bash
python3 script.py suspend
```

* 가상 머신 재개
```bash
python3 script.py resume
```

* 가상 머신 스냅샷 생성
```bash
python3 script.py snapshot
```
