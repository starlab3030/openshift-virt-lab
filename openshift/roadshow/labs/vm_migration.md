# 가상머신 마이그레이션

목차
1. [가상머신 마이그레이션 소개](./vm_migration.md#1-가상머신-마이그레이션-소개)<br>
2. [VMware 프로바이더를 위한 사전 준비](./vm_migration.md#2-vmware-프로바이더를-위한-사전-준비)<br>
3. [VMware로부터 가상머신 마이그레이션](./vm_migration.md#3-vmware로부터-가상머신-마이그레이션)<br>
   3.1 [VMware 환경 리뷰](./vm_migration.md#31-vmware-환경-리뷰)<br>
   3.2 [마이그레이션 툴킷에 대한 VMware 공급자 생성](./vm_migration.md#32-마이그레이션-툴킷에-대한-vmware-공급자-생성)<br>
   3.3 [스토리지 및 네트워크 매핑 생성](./vm_migration.md#33-스토리지-및-네트워크-매핑-생성)<br>
   3.4 [마이그레이션 계획 생성](./vm_migration.md#34-마이그레이션-계획-생성)<br>
   3.5 [마이그레이션된 가상머신 리뷰 및 구성](./vm_migration.md#35-마이그레이션-가상머신-리뷰-및-구성)<br>
4. [요약](./vm_migration.md#4-요약)
<br>
<br>

## 1. 가상머신 마이그레이션 소개

이 실습에서는 MTV(Migration Toolkit for Virtualization)를 사용하여 VMware vSphere에서 오픈시프트로 가상머신을 가져옵니다. 마이그레이션 툴킷은 두 가지 가져오기 "모드"를 지원합니다.

* 콜드(cold) 마이그레이션은 마이그레이션을 시작하기 전에 소스 가상머신을 끕니다. 이것이 기본 마이그레이션 유형입니다.
* 웜(warm) 마이그레이션은 원본 가상머신이 계속 실행되는 동안 데이터를 복사합니다. 대량의 데이터가 마이그레이션되면 가상머신이 종료되고 최종 데이터가 대상에 복사됩니다. 그런 다음 새 가상머신을 시작할 수 있으므로 가상머신 호스팅 애플리케이션의 가동 중지 시간이 훨씬 단축됩니다.

마이그레이션 도구 키트는 이미 오퍼레이터를 사용하여 클러스터에 배포되었습니다. 오퍼레이터 설치 및 구성 방법에 대한 문서는 [여기](https://access.redhat.com/documentation/en-us/migration_toolkit_for_virtualization/)에서 찾을 수 있습니다.

가상화용 마이그레이션 툴킷을 구성하는 방법에 대해 자세히 알아보려면 여기에서 레드햇 가상화 관련 [문서](https://access.redhat.com/documentation/en-us/migration_toolkit_for_virtualization/2.4/html/installing_and_using_the_migration_toolkit_for_virtualization/prerequisites#rhv-prerequisites_mtv)를 참조하거나 VMware vSphere에 대하여 [여기](https://access.redhat.com/documentation/en-us/migration_toolkit_for_virtualization/2.4/html/installing_and_using_the_migration_toolkit_for_virtualization/prerequisites#vmware-prerequisites_mtv)에서 알아보십시오.
<br>
<br>

## 2. VMware 프로바이더를 위한 사전 준비

모든 마이그레이션에는 다음 필수 구성 요소가 적용됩니다.

* ISO/CD-ROM 디스크는 마운트 해제되어야 합니다.
* 각 NIC에는 하나의 IPv4 및/또는 하나의 IPv6 주소가 포함되어야 합니다.
* 가상머신 운영체제는 [오픈시프트 가상화에서 게스트 운영체제](https://access.redhat.com/articles/973163#ocpvirt)로 사용하기 위해 인증 및 지원되어야 합니다.
* 가상머신 이름에는 소문자(a\~z), 숫자(0\~9) 또는 하이픈(-)만 포함해야 하며 최대 253자까지 가능합니다. 첫 번째와 마지막 문자는 영숫자여야 합니다. 이름에는 대문자, 공백, 마침표(.) 또는 특수 문자가 포함될 수 없습니다.
* 가상머신 이름은 오픈시프트 가상화 환경의 기존 가상머신 이름과 중복되어서는 안 됩니다.

**Migration Toolkit for Virtualization**은 규칙을 준수하지 않는 가상머신에 자동으로 새 이름을 할당합니다. 이런 일이 발생하면 MTV는 마이그레이션이 원활하게 진행될 수 있도록 자동으로 새로운 가상머신 이름을 생성합니다.
<br>
<br>

## 3. VMware로부터 가상머신 마이그레이션

오픈시프트로 마이그레이션할 수 있도록 3계층 애플리케이션이 VMware에 배포되었습니다.

애플리케이션은 다음 4개의 가상머신으로 구성됩니다.

* 트래픽을 웹 서버로 리디렉션하는 단일 프록시 시스템 (1개의 가상머신))
* 데이터베이스에 연결되는 PHP 애플리케이션을 호스팅하는 IIS가 있는 두 개의 마이크로소프트 윈도우 서버 (2개의 가상머신)
* MariaDB 데이터베이스를 실행하는 하나의 리눅스 시스템 (1개의 가상머신)

이 애플리케이션은 다음 링크에서 액세스할 수 있습니다: http://webapp.vc.opentlc.com/

4개의 가상 머신 중 3개를 마이그레이션합니다. 오픈시프트는 SDN에 연결된 가상머신에 대한 네트워크 트래픽 및 로드 밸런싱을 기본적으로 처리하므로 프록시(로드 밸런서) 가상머신을 마이그레이션할 필요가 없습니다.

<br>

### 3.1 VMware 환경 리뷰

1. https://portal.vc.opentlc.com으로 이동합니다.

<br>

2. 사용자 `%vcenter_user%@vc.opentlc.com` 및 비밀번호 `%vcenter_password%`로 로그인합니다.

   <img src="lab-images/vm_migration--3.1.2_vSphere_VM_List.png" title="100px" alt="vSphere의 가상머신 리스트"></img> <br> 

> [!NOTE]
> 접미사가 `_running`인 가상머신이 활성화된 가상머신입니다. 마이그레이션을 중지해야 하는 경우 마이그레이션을 위해 가상머신의 복제본이 생성되었습니다. 해당 가상머신은 해당 접미사가 없는 가상머신입니다.
<br>

3. 이 [링크](https://portal.vc.opentlc.com/ui/app/dvportgroup;nav=n/urn:vmomi:DistributedVirtualPortgroup:dvportgroup-1916:ee1bef3e-6179-4c1f-9d2a-004c7b0df4e5/ports)로 이동하여 세그먼트를 검토하세요.

   <img src="lab-images/vm_migration--3.1.3_vSphere_Network.png" title="100px" alt="vSphere의 네트워크"></img> <br> 
<br>

4. 이 [링크](https://portal.vc.opentlc.com/ui/app/datastore;nav=s/urn:vmomi:Datastore:datastore-48:ee1bef3e-6179-4c1f-9d2a-004c7b0df4e5/vms/vms)로 이동하여 스토리지를 검토하세요.

   <img src="lab-images/vm_migration--3.1.4_vSphere_Datastore.png" title="100px" alt="vSphere의 데이터 소스"></img> <br> 
<br>

### 3.2 마이그레이션 툴킷에 대한 VMware 공급자 생성

MTV(Migration Toolkit for Virtualization)는 VMware 가상 디스크 개발 키트(VDDK:Virtual Disk Development Kit)) SDK를 사용하여 VMware vSphere에서 가상 디스크를 전송합니다. 이 VDDK는 이 환경에 이미 설정되어 있습니다.
<br>

1. 왼쪽 메뉴에서 **마이그레이션(Migration)** → **가상화 제공업체(Providers for virtualization)** 로 이동합니다.

<br>

2. 프로젝트 `openshift-mtv`를 선택하세요.

<br>

3. 기본적으로 **OpenShift Virtualization**을 대상 플랫폼으로 나타내는 `호스트(host)`라는 공급자가 있습니다.
   
   <img src="lab-images/vm_migration--3.2.3_MTV_Provider_list.png" title="100px" alt="MTV 프로바이더 리스트 확인"></img> <br> 
   소스 vCenter 시스템을 Migration Toolkit for Virtualization에 새 공급자로 등록해야 합니다.
<br>

4. 오른쪽 상단의 **Create Provider** 버튼을 누릅니다. 대화 상자가 나타납니다.

   <img src="lab-images/vm_migration--3.2.4_MTV_Create_Provider.png" title="100px" alt="MTV 프로바이더 생성"></img> <br> 
<br>

5. **공급자 유형(Provider type)** 드롭다운에서 **VMware**를 선택하고 다음 데이터를 입력합니다.

   <img src="lab-images/vm_migration--3.2.5_MTV_Fill_Dialog.png" title="100px" alt="VMware 용 프로바이더 설정"></img> <br> 

   1. **Provider Name**: `vmware`
   2. **vCenter server hostname or IP address**: `portal.vc.opentlc.com`
   3. **vCenter user name**: `%vcenter_user%@vc.opentlc.com`
   4. **vCenter password**: `%vcenter_password%`
   5. **VDDK init image**: `image-registry.openshift-image-registry.svc:5000/openshift/vddk:latest`
   6. **SHA-1 fingerprint**: `C7:BF:C2:DD:CD:73:1C:22:DC:D1:5A:DD:EA:64:21:C1:97:FB:F0:9C`
<br>

6. **Create**를 누르고 **Status** 열이 `Ready`로 변경될 때까지 기다립니다.

   <img src="lab-images/vm_migration--3.2.6_MTV_Provider_Added.png" title="100px" alt="생성된 프로바이더 확인"></img> <br> 
   이제 MTV는 귀하의 VMware vSphere 환경을 알고 연결할 수 있습니다.
<br>

### 3.3 스토리지 및 네트워크 매핑 생성

VMware vSphere와 레드햇 오픈시프트에서는 스토리지와 네트워킹이 다르게 관리됩니다. 따라서 VMware vSphere의 소스 데이터 저장소 및 네트워크에서 오픈시프트의 해당 항목에 대한 (간단한) 매핑을 생성해야 합니다. 그런 다음 이 매핑은 VMware vSphere 네트워크 및 스토리지 정의를 오픈시프트 네트워크 및 스토리지 정의로 변환하는 데 사용됩니다.

이는 한 번만 구성하면 이후 가상머신 마이그레이션 계획에서 재사용됩니다.
<br>

1. 왼쪽 메뉴에서 **Migration** → **NetworkMaps for virtualization**로 이동한 후 **Create NetworkMap**을 누릅니다.

   <img src="lab-images/vm_migration--3.3.1_MTV_NetworkMaps.png" title="100px" alt="MTV 네트워크 맵"></img> <br>
<br>

2. 나타나는 대화상자에 다음 정보를 입력하세요. **만들기(Create)** 를 누릅니다.

   <img src="lab-images/vm_migration--3.3.2_Add_VMWARE_Mapping_Network.png" title="100px" alt="VMware 네트워크 매핑 추가"></img> <br>
<br>

3. 생성된 매핑이 `준비(Ready)` **상태(Status)** 인지 확인합니다.

   <img src="lab-images/vm_migration--3.3.3_List_VMWARE_Mapping_Network.png" title="100px" alt="VMware 네트워크 매핑 리스트"></img> <br>
<br>

4. 왼쪽 메뉴에서 **Migration** → **StorageMaps for virtualization**로 이동한 후 **Create StorageMap**을 누릅니다.

   <img src="lab-images/vm_migration--3.3.4_MTV_StorageMaps.png" title="100px" alt="MTV 스토리지 맵"></img> <br>
<br>

5. 다음 정보를 입력 후 **만들기(Create)** 를 누릅니다.

   <img src="lab-images/vm_migration--3.3.5_Add_VMWARE_Mapping_Storage.png" title="100px" alt="VMware 스토리지 매핑 추가"></img> <br>

   1. **Name**: `mapping-datasource`
   2. **Source provider**: `vmware`
   3. **Target provider**: `host`
   4. **Source storage**: `WorkloadDatastore`
   5. **Target storage class**: `ocs-storagecluster-ceph-rbd (default)`
<br>

6. 생성된 매핑이 `준비(Ready)` **상태(Status)** 인지 확인합니다.

   <img src="lab-images/vm_migration--3.3.6_List_VMWARE_Mapping_Storage.png" title="100px" alt="VMware 스토리지 매핑 리스트"></img> <br>
<br>

### 3.4 마이그레이션 계획 생성

이제 가상화 공급자와 두 가지 매핑(네트워크 및 스토리지)이 있으므로 마이그레이션 계획을 생성할 수 있습니다. 이 계획에서는 VMware vSphere에서 레드햇 오픈시프트 가상화로 마이그레이션할 가상머신과 마이그레이션 실행 방법(콜드/웜, 네트워크 매핑, 스토리지 매핑, 사전/사후 후크 등)을 선택합니다.

1. 왼쪽 메뉴에서 **마이그레이션(Migration)** → **가상화 계획(Plans for virtualization)** 으로 이동한 후 **계획 생성(Create plan)** 을 누릅니다.

   <img src="lab-images/vm_migration--3.4.1_Create_VMWARE_Plan.png" title="100px" alt="VMware 마이그레이션 계획 생성"></img> <br>
<br>

2. 마법사의 **일반 설정(General settings)** 단계에서 다음 정보를 입력합니다. 완료되면 **다음(Next)** 을 누르세요.

   <img src="lab-images/vm_migration--3.4.2_General_VMWARE_Plan.png" title="100px" alt="VMware 마이그레이션 일반 계획"></img> <br>

   1. **Plan name**: `move-webapp-vmware`
   2. **Source provider**: `vmware`
   3. **Target provider**: `host`
   4. **Target namespace**: `vmexamples`
<br>

3. 다음 단계에서 **모든 데이터 센터(All datacenters)** 를 선택하고 **다음(Next)** 을 누릅니다.

   <img src="lab-images/vm_migration--3.4.3_VM_Filter_VMWARE_Plan.png" title="100px" alt="VMware 마이그레이션 계획 가상머신 필터"></img> <br>
<br>

4. 다음 단계에서는 모든 가상머신을 선택하고 **다음(Next)** 을 누르세요:

   <img src="lab-images/vm_migration--3.4.4_VM_Select_VMWARE_Plan.png" title="100px" alt="VMware 마이그레이션 계획 가상머신 선택"></img> <br>
<br>

5. **네트워크 매핑(Network mapping)** 단계에서 `매핑-세그먼트(mapping-segment)`를 선택하고 **다음(Next)** 을 누릅니다.

   <img src="lab-images/vm_migration--3.4.5_Network_VMWARE_Plan.png" title="100px" alt="VMware 마이그레이션 계획 네트워크"></img> <br>
<br>

6. **스토리지 매핑(Storage mapping)** 단계에서 `매핑-데이터 저장소(mapping-datastore)`를 선택하고 **다음(Next)** 을 누릅니다.

   <img src="lab-images/vm_migration--3.4.6_Storage_VMWARE_Plan.png" title="100px" alt="VMware 마이그레이션 계획 스토리지"></img> <br>
<br>

7. **Type** 및 **Hooks** 단계에서 **다음(Next)** 를 누릅니다.

<br>

8. 지정된 구성을 검토하고 **마침(Finish)** 을 누릅니다.

   <img src="lab-images/vm_migration--3.4.8_Finish_VMWARE_Plan.png" title="100px" alt="VMware 마이그레이션 계획 종료"></img> <br>
<br>

9. 계획 상태가 `준비(Ready)`인지 확인하세요.

   <img src="lab-images/vm_migration--3.4.9_Ready_VMWARE_Plan.png" title="100px" alt="VMware 마이그레이션 계획 준비"></img> <br>
<br>

10. **시작(Start)** 을 눌러 세 개의 가상머신에 대하여 마이그레이션을 **시작(Start)** 합니다.

<br>

11. 약 10분 정도 지나면 마이그레이션이 완료됩니다.

    <img src="lab-images/vm_migration--3.4.11_Completed_VMWARE_Plan.png" title="100px" alt="VMware 마이그레이션 계획 완료"></img> <br>

> [!IMPORTANT]
> 많은 참가자가 동일한 작업을 병렬로 수행하면 이 작업이 실제 환경보다 느리게 수행될 수 있습니다. 기다려주십시오.
<br>

### 3.5 마이그레이션 가상머신 리뷰 및 구성

이제 가상머신이 마이그레이션되었으며 오픈시프트 가상화에서 시작할 수 있습니다. VMware vCenter에서와 마찬가지로 가상머신 콘솔에 연결하고 상호 작용할 수 있습니다.

그러나 가상머신은 아직 서로 연결되지 않았습니다. 이는 가상머신을 SDN에 연결했기 때문입니다. 이는 가상머신이 이전에 연결되었던 외부 네트워크와 약간 다르게 작동합니다.

오픈시프트의 로드밸런서를 **서비스(Service)** 라고 합니다. 로드밸런싱 서비스는 대상에 할당된 라벨을 통해 부하 분산하는 트래픽의 수신자를 선택합니다. 현재 가상머신에는 아직 라벨이 할당되지 않았습니다.

가상머신을 서비스와 성공적으로 연결하려면 다음을 수행해야 합니다.

* 가상머신에 라벨을 추가합니다. 두 윈도우 IIS 서버 모두 동일한 로드밸런서 뒤에 있으므로 동일한 레이블을 사용합니다.
* 두 개의 윈도우 IIS 서버를 클러스터의 다른 작업에 사용할 수 있도록 서비스를 만듭니다. 오픈시프트는 서비스 이름을 DNS 이름으로 사용하여 내부적으로 로드밸런서에 자동으로 액세스할 수 있도록 합니다.
* **경로(Route)** 를 생성하여 오픈시프트 외부에서 서비스를 사용할 수 있도록 합니다.

<br>

1. 오픈시프트 콘솔에서 **가상화(Virtualization)** → **VirtualMachines**로 이동하여 마이그레이션된 가상머신을 성공적으로 가져오고 실행 중인지 확인합니다.

   <img src="lab-images/vm_migration--3.5.1_VMWARE_VMs_List.png" title="100px" alt="VMware로부터 마이그레이션 된 가상머신 리스트 확인"></img> <br>

> [!NOTE]
> `vmexamples` 프로젝트를 선택했는지 확인하세요.
<br>

2. `winweb01`에 액세스하고 **YAML** 탭으로 이동합니다.

<br>

3. `spec:` 섹션을 찾고 `template.metadata` 아래에 다음 줄을 추가하여 가상머신 리소스에 레이블을 지정합니다.

   ```yaml
         labels:
           env: webapp
   ```
   
   <img src="lab-images/vm_migration--3.5.3_VMWARE_VMs_YAML.png" title="100px" alt="VMware로부터 마이그레이션 된 가상머신 YAML 파일 확인"></img> <br>

> [!IMPORTANT]
> 위 스크린샷처럼 들여쓰기가 정확히 맞는지 확인하세요.
<br>

4. 가상머신 `winweb02`에 대해 프로세스를 반복합니다.

<br>

5. *가상 머신(Virtual Machines)* `database`, `winweb01` 및 `winweb02`를 시작합니다.

   * 각 가상머신의 콘솔 탭에 액세스하여 가상머신이 제대로 작동하는지 확인하세요.
<br>

6. **네트워킹(Networking)** → **서비스(Services)** 로 이동하고 **서비스 생성(Create Service)** 을 누릅니다. 가상머신에 추가한 레이블(`env=webapp`)을 기억하시나요? 여기서는 서비스가 선택기(selector)에서 해당 레이블을 사용하여 트래픽을 라우팅할 가상머신을 선택하는 것을 볼 수 있습니다.

<br>

7. YAML을 다음 정의로 바꿉니다.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: webapp
     namespace: vmexamples
   spec:
     selector:
       env: webapp
     ports:
       - protocol: TCP
         port: 80
         targetPort: 80
   ```
<br>

8. **생성(Create)** 을 누릅니다.

<br>

9. 이제 오픈시프트 클러스터 내에서 윈도우 IIS에 액세스할 수 있습니다. 다른 가상머신은 DNS 이름 `webapp.vmexamples`를 사용하여 여기에 액세스할 수 있습니다. 그러나 이러한 웹 서버는 외부에서 액세스 가능한 애플리케이션의 프런트 엔드이므로 **경로(Route)** 를 사용하여 노출할 것입니다.

   왼쪽 탐색 메뉴에서 **네트워킹(Networking)** → **경로(Routes)** 로 이동합니다. **경로 생성(Create Route)** 을 누르고 다음 정보를 입력하세요.

   1. Name: `route-webapp`
   2. Service: `webapp`
   3. Target port: `80 → 80 (TCP)`
<br>

10. **만들기(Create)** 를 누르세요

    <img src="lab-images/vm_migration--3.5.10_VMWARE_VMs_Create_Route.png" title="100px" alt="마이그레이션 된 가상머신 경로 생성"></img> <br>

> [!NOTE]
> 오픈시프트는 경로를 통해 클러스터에 들어오는 트래픽을 자동으로 (재)암호화할 수 있지만 이 애플리케이션에는 TLS를 사용할 필요가 없습니다. **보안 경로(Secure Route)** 옵션을 선택하면 안 됩니다.
<br>

11. **위치(Location)** 필드에 표시된 주소로 이동합니다.

    <img src="lab-images/vm_migration--3.5.11_VMWARE_VMs_URL.png" title="100px" alt="가상머신의 경로 URL 확인"></img> <br>
<br>

12. 페이지가 로드되면 오류가 표시됩니다. 이는 윈도우 웹 서버가 데이터베이스 가상머신에 연결하기 위해 내부 이름 `database`를 확인할 수 없기 때문입니다.

    연결 문제를 해결하려면 SDN에 연결된 다른 가상머신에서 검색할 수 있도록 데이터베이스 가상머신에 대한 또 다른 서비스를 생성해야 합니다. 이 데이터베이스는 오픈시프트 환경 외부에서 액세스할 필요가 없으므로 이 서비스에 대한 경로를 생성할 필요가 없습니다.

    **네트워킹(Networking)** → **서비스(Services)** 로 이동하여 **서비스 생성(Create service)** 을 누르세요. YAML을 다음 정의로 바꿉니다.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: database
      namespace: vmexamples
    spec:
      selector:
        vm.kubevirt.io/name: database
      ports:
        - protocol: TCP
          port: 3306
          targetPort: 3306    
    ```

> [!NOTE]
> 이 예에서 서비스는 단순히 가상머신 이름 선택기(selector)를 사용하고 있습니다. 모든 가상머신에 자동으로 추가되는 기본 레이블입니다. 선택기(selector)와 일치하는 가상머신이 하나만 있기 때문에 서비스는 데이터베이스에 대한 부하를 분산하지 않고 대신 내부 DNS 이름을 통한 검색을 위해 서비스를 사용합니다.
<br>

13. webapp URL을 다시 로드하면 적절한 결과를 얻을 수 있습니다.

    <img src="lab-images/vm_migration--3.4.11_Completed_VMWARE_Plan.png" title="100px" alt="webapp의 URL을 리로드하여 결과 확인"></img> <br>
<br>
<br>

## 4. 요약

가상화용 마이그레이션 도구 키트 (Migration Toolkit for Virtualization) 외에도 세 가지 다른 마이그레이션 도구 키트가 있습니다. 이러한 조합을 사용하면 조직의 요구 사항에 따라 오픈시프트 클러스터 내에서 많은 워크로드를 이동할 수 있습니다.

* [Migration Toolkit for Runtimes](https://developers.redhat.com/products/mtr/overview) - 자바 애플리케이션 현대화 및 마이그레이션을 지원하고 가속화합니다.
* [Migration Toolkit for Applications](https://access.redhat.com/documentation/en-us/migration_toolkit_for_applications/) - 컨테이너 및 쿠버네티스에 대한 대규모 애플리케이션 현대화 노력을 가속화합니다.
* [Migration Toolkit for Containers](https://docs.openshift.com/container-platform/4.12/migration_toolkit_for_containers/about-mtc.html) - 오픈시프트 클러스터 간에 상태 저장 애플리케이션 워크로드를 마이그레이션합니다.

이에 대한 자세한 내용은 레드햇 고객 담당 팀에 문의하세요.

이 모듈에서는 VMware vSphere에서 레드햇 오픈시프트 가상화로 가상머신을 마이그레이션하는 방법을 살펴보았습니다. 두 개의 윈도우 시스템과 하나의 리눅스 시스템을 포함하는 웹 애플리케이션을 마이그레이션했습니다. 오픈시프트 기능을 사용하여 애플리케이션에 대한 네트워킹 액세스를 제공했으며, 프로젝트 내부 액세스를 제공하는 서비스를 생성하는 방법을 배웠습니다.
<br>
<br>

------
[차례](../README.md) &nbsp;&nbsp;&nbsp;&nbsp; [<< 스토리지 관리 <<](./storage_management.md) &nbsp;&nbsp;&nbsp;&nbsp; [>> 가상머신 로드밸런싱 >>](./vm_load_balancing.md)