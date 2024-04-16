# 가상머신 로드-밸런싱

목차
1. [레이어 2 모드](./vm_load_balancing.md#1-레이어-2-모드)<br>
2. [레이어 3 모드](./vm_load_balancing.md#2-레이어-3-모드)<br>
3. [오퍼레이터 리뷰](./vm_load_balancing.md#3-오퍼레이터-리뷰)<br>
   3.1 [IP AddressPool](./vm_load_balancing.md#31-ip-addresspool)<br>
   3.2 [레이어 2 모드 구성](./vm_load_balancing.md#32-레이어-2-모드-구성)<br>
4. [데이터베이스 노드 외부 노출하기](./vm_load_balancing.md#4-데이터베이스-노드-외부-노출하기)<br>
5. [요약](./vm_load_balancing.md#5-요약)<br>

<br>

이 실습에서는 MetalLB 오퍼레이터를 검토하고 클러스터 외부에 서비스를 노출합니다.
<br>

**목표**
* 오퍼레이터 리뷰
* 서비스를 외부에 노출
<br>

MetalLB를 사용하는 것은 베어메탈 클러스터 또는 베어메탈처럼 처리되는 가상 인프라가 있고 외부 IP 주소를 통해 애플리케이션에 대한 내결함성 액세스를 보장하려는 경우 유용합니다.

MetalLB가 이러한 요구 사항을 충족하려면 외부 IP 주소에 대한 네트워크 트래픽이 클라이언트에서 클러스터의 호스트 네트워크로 라우팅되도록 네트워킹 인프라를 구성해야 합니다.

두 가지 모드로 작동할 수 있습니다.

* **레이어2 모드에서 작동하는 MetalLB**
  + IP 장애 조치와 유사한 메커니즘을 활용하여 장애 조치를 지원
  + 그러나 가상 라우터 중복 프로토콜(VRRP: Virtual Router Redundancy Protocol) 및 연결 유지에 의존하는 대신 MetalLB는 가십 기반 프로토콜을 활용하여 노드 오류 인스턴스를 식별
  + 오류가 감지되면 다른 노드가 리더 노드의 역할을 맡고 이 변경 사항을 브로드캐스트하기 위해 무료 ARP 메시지가 발송
<br>

* **레이어3 또는 BGP(Border Gateway Protocol) 모드**
  + 여기서 작동하는 MetalLB는 오류 감지를 네트워크에 위임
  + **오픈시프트** 노드가 연결 설정한 BGP 라우터는 모든 노드 오류를 식별하고 해당 노드에 대한 경로를 종료

Pod 및 서비스의 고가용성을 보장하려면 IP 장애 조치 대신 MetalLB를 사용하는 것이 좋습니다.
<br>
<br>

## 1. 레이어 2 모드

레이어 2 모드에서는 한 노드의 스피커(speaker) 포드가 서비스의 외부 IP 주소를 호스트 네트워크에 알립니다. 네트워크 관점에서 볼 때 노드에는 네트워크 인터페이스에 할당된 여러 IP 주소가 있는 것으로 보입니다.

레이어 2 모드에서는 서비스 IP 주소에 대한 모든 트래픽이 하나의 노드를 통해 라우팅됩니다. 트래픽이 노드에 진입한 후 CNI 네트워크 공급자의 서비스 프록시는 해당 서비스의 모든 Pod에 트래픽을 배포합니다.

노드를 사용할 수 없게 되면 장애 조치가 자동으로 수행됩니다. 다른 노드의 스피커(speaker) 포드는 노드를 사용할 수 없음을 감지하고, 살아남은 노드의 새 스피커(speaker) 포드는 실패한 노드의 서비스 IP 주소 소유권을 가져옵니다.

<img src="lab-images/vm_lb--0.1_layer2.png" title="100px" alt="가상머신 로드밸런싱 레이어 2 모드"></img> <br> 
<br>
<br>

## 2. 레이어 3 모드

BGP 모드에서는 기본적으로 각 스피커(speaker) 포드가 서비스의 로드 밸런서 IP 주소를 각 BGP 피어(peer)에 알립니다. 선택적 BGP 피어(peer) 목록을 추가하여 특정 풀에서 들어오는 IP를 특정 피어(peer) 집합에 알리 수도 있습니다. BGP 피어(peer)는 일반적으로 BGP 프로토콜을 사용하도록 구성된 네트워크 라우터입니다. 라우터가 로드 밸런서 IP 주소에 대한 트래픽을 수신하면 라우터는 IP 주소를 광고한 스피커(speaker) 포드가 있는 노드 중 하나를 선택합니다. 라우터는 해당 노드로 트래픽을 보냅니다. 트래픽이 노드에 진입한 후 CNI 네트워크 플러그인의 서비스 프록시는 서비스의 모든 Pod에 트래픽을 배포합니다.

노드를 사용할 수 없게 되면 라우터는 로드 밸런서 IP 주소를 광고하는 스피커 포드가 있는 다른 노드와의 새 연결을 시작합니다.

<img src="lab-images/vm_lb--0.2_bgp.png" title="100px" alt="가상머신 로드밸런싱 BGP 모드"></img> <br> 
<br>
<br>

## 3. 오퍼레이터 리뷰

1. **오퍼레이터(Operators)** → **설치된 오퍼레이터(Installed Operators)** 로 이동합니다. **모든 프로젝트(All Projects)**를 선택하고 **MetalLB**를 선택합니다.

   <img src="lab-images/vm_lb--3.0.1_Operator_Installed.png" title="100px" alt="가상머신 로드밸런싱 오퍼레이터 설치"></img> <br>
<br>

2. 세부정보 탭에서 **제공된(Provided) APIs**를 검토하세요.

   <img src="lab-images/vm_lb--3.0.2_Review_Operator.png" title="100px" alt="가상머신 로드밸런싱 오퍼레이터 리뷰"></img> <br>
<br>

3. **MetalLB** 탭을 선택하여 배포가 올바르게 설치 및 구성되었는지 확인합니다.

   <img src="lab-images/vm_lb--3.0.3_Review_Operator_MetalLB.png" title="100px" alt="가상머신 로드밸런싱 MetalLB 리뷰"></img> <br>
<br>

### 3.1 IP AddressPool

이 랩에서는 오픈시프트 클러스터 노드가 있는 동일한 네트워크(`192.168.123.0/24`)를 사용하고,  이를 위해 오픈시프트 클러스터에서 IP 범위 `192.168.123.200-192.168.123.250`을 로드밸런싱 서비스에 사용하도록 예약합니다.

1. 프로젝트 `metallb-system`으로 전환

<br>

2. **IPAddressPool** 탭으로 전환하고 **Create IPAddressPool**을 누릅니다.

   <img src="lab-images/vm_lb--3.1.2_MetalLB_IPAddressPool_Form.png" title="100px" alt="MetalLB IPAddressPool 폼"></img> <br> 
<br>

3. `ip-addresspool-webapp` 이름을 사용하고 *주소(addresses)* 섹션 아래에서 기존 주소를 모두 제거하고 `192.168.123.200-192.168.123.250`을 주소 풀로 구성합니다. 완료되면 다음 이미지와 유사하게 보일 것입니다.

   <img src="lab-images/vm_lb--3.1.3_MetalLB_IPAddressPool_Defined.png" title="100px" alt="MetalLB IPAddressPool 정의"></img> <br> 
<br>

4. 아래로 스크롤하여 **만들기(Create)** 를 누릅니다.

<br>

### 3.2 레이어 2 모드 구성

이를 위해 실습에서는 Layer2 모드에서 MetalLB를 사용하겠습니다.

1. **L2Advertisement** 탭으로 전환하고 **Create L2Advertisement**을 누릅니다.

<br>

2. `l2-adv-webapp` 이름을 표시하고 *ipaddressPools* 섹션에서 다음과 같이 `ip-addresspool-webapp` 값을 지정합니다.

   <img src="lab-images/vm_lb--3.2.2_MetalLB_L2Advertisement.png" title="100px" alt="MetalLB L2Advertisement"></img> <br>
<br>

3. **Create** 를 누르세요

<br>
<br>

## 4. 데이터베이스 노드 외부 노출하기

VMWare에서 마이그레이션된 가상머신은 현재 이전 모듈에서 생성된 서비스를 사용하여 클러스터 내부에서 액세스할 수 있습니다. 이 작업에서는 클러스터 외부에 포트 3306을 노출합니다.

1. **네트워킹(Networking)** → **서비스(Services)** 로 이동하여 `vmexamples` 프로젝트를 선택합니다.

   <img src="lab-images/vm_lb--4.1_Services.png" title="100px" alt="가상머신 로드밸런싱 서비스"></img> <br>
<br>

2. **서비스 생성(Create Service)** 을 누르고 다음 코드 조각으로 양식을 작성합니다.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: database-metallb
     namespace: vmexamples
   spec:
     type: LoadBalancer
     selector:
       vm.kubevirt.io/name: database
     ports:
       - protocol: TCP
         port: 3306
         targetPort: 3306   
   ```

> [!NOTE]
> 표시된 `유형(type)`은 `LoadBalancer`입니다. 이 클러스터에는 MetalLB가 설치되어 있으므로 이를 사용하여 지정된 포트가 노출됩니다. F5, Nginx 등과 같은 파트너가 제공하는 다른 로드밸런서 옵션이 있습니다.
<br>

3. **생성(Create)** 을 누르고 생성된 **서비스(Service)** 를 검토합니다. 로드밸런서에 할당된 IP 주소는 이전 실습에서 지정한 범위에 속합니다.

   <img src="lab-images/vm_lb--4.3_Service_created.png" title="100px" alt="가상머신 로드밸런싱 생성된 서비스"></img> <br>
<br>

4. 외부 IP를 통해 데이터베이스 서비스에 대한 연결을 확인하려면 오른쪽 상단의 다음 아이콘을 클릭하여 웹 터미널을 엽니다.

   <img src="lab-images/vm_lb--4.4_OCP_Terminal_Icon.png" title="100px" alt="가상머신 콘솔 아이콘"></img> <br>
<br>

5. 화면 하단에 콘솔이 나타납니다

   <img src="lab-images/vm_lb--4.5_OCP_Terminal.png" title="100px" alt="가상머신 콘솔 연결"></img> <br>
<br>

6. 오른쪽 콘솔을 사용하여 할당된 IP와 포트 3306에 액세스해 봅니다.

   ```bash
   [~] $ curl -s 192.168.123.202:3306 | cut -c1-16
   ```

   출력 예
   ```bash
   5.5.68-MariaDB
   ```
<br>
<br>

## 5. 요약

MetalLB는 NMstate 또는 멀터스를 사용하여 물리적 네트워크를 구성할 필요 없이 클러스터 외부에 애플리케이션을 노출하는 베어메탈 온프레미스 배포를 위한 간단하고 간단한 솔루션입니다.
<br>
<br>

------
[차례](../README.md) &nbsp;&nbsp;&nbsp;&nbsp; [<< 가상머신 마이그레이션 <<](./vm_migration.md) &nbsp;&nbsp;&nbsp;&nbsp; [>> 백업 및 복구 >>](./backup_and_restore.md)