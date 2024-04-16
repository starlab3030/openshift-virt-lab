# 스토리지 관리

목차
1. [스토리지 관리 소개](./storage_management.md#1-스토리지-관리-소개)<br>
2. [오픈시프트 가상화 기본 부팅 소스](./storage_management.md#2-오픈시프트-가상화-기본-부팅-소스)<br>
3. [스냅샷](./storage_management.md#3-스냅샷)<br>
   3.1 [스냅샷 소개](./storage_management.md#31-스냅샷-소개)<br>
   3.2 [스냅샷으로 작업](./storage_management.md#32-스냅샷으로-작업)<br>
4. [가상머신 복제](./storage_management.md#4-가상머신-복제)<br>
5. [요약](./storage_management.md#5-요약)<br>
   5.1 [용어집](./storage_management.md#51-용어집)
<br>
<br>

## 1. 스토리지 관리 소개

레드햇 오픈시프트는 온프레미스 및 클라우드 공급자 모두를 위해 다양한 유형의 스토리지를 지원합니다. 오픈시프트 가상화는 실행 중인 환경에서 지원되는 모든 *컨테이너 스토리지 인터페이스(CSI: Container Storage Interface)* 프로비저너를 사용할 수 있습니다. 예를 들어 OpenShift Data Foundation, NetApp, Dell/EMC, Fujitsu, Hitachi, Pure Storage, Portworx 등은 오픈시프트 가상화를 통해 온프레미스, CSI 프로비저닝, ReadWriteMany(RWX) 볼륨을 지원합니다.

이 워크숍에서는 공급자에게 스토리지를 요청하고 가상머신 디스크를 저장하는 데 사용되는 영구 볼륨 클레임 (PVCs: Persistent Volume Claims)을 살펴봅니다. 많은 스토리지 제공업체는 해당 장치의 스냅샷 및 복제도 지원합니다. CSI 드라이버 및 스토리지 장치에서 지원하는 기능을 확인하려면 공급업체에 문의하세요.

특히 오픈시프트 가상화와 관련된 스토리지 프로토콜(예: NFS, iSCSI, FC 등)에는 제한이 없습니다. 유일한 요구 사항은 실시간 마이그레이션에 RWX 액세스 모드를 사용할 수 있어야 한다는 것입니다. 그렇지 않은 경우에는 가상머신 및 애플리케이션의 요구 사항을 가장 잘 충족하는 스토리지가 항상 올바른 선택입니다.

<img src="lab-images/storage_mgmt--1.0_disk_concepts.png" title="100px" alt="스토리지 관리 디스크 개념"></img> <br> 
<br>
<br>

## 2. 오픈시프트 가상화 기본 부팅 소스

영구 가상 머신에는 영구 스토리지가 필요합니다. 이 랩 환경에는 애플리케이션 데이터 및 가상머신 디스크를 호스팅하기 위한 공유 영구 볼륨에 대한 액세스를 제공하기 위해 OpenShift Data Foundation이 배포되어 있습니다. 오픈시프트 가상화 오퍼레이터를 설치하는 동안 다양한 리눅스 배포용 템플릿 디스크를 보관하기 위해 일부 `PertantVolumeClaim`이 자동으로 생성되었습니다. 여기에는 다음이 포함됩니다.
   * Red Hat Enterprise Linux 8
   * Red Hat Enterprise Linux 9
   * Fedora
   * CentOS 7
   * CentOS Stream 8
   * CentOS Stream 9

이러한 운영체제 이미지 사용은 선택 사항이며, 오퍼레이터의 적절한 설정을 통해 생성 및 다운로드를 비활성화할 수 있습니다. 각 배포에 대해 "클라우드" 이미지를 사용하는 것은 오픈시프트 가상화로 가상머신 프로비저닝을 시작하는 빠르고 편리한 방법입니다. 이 이미지에는 cloud-init가 포함되어 있으며 가상화에 최적화된 축소된 용량의 운영체제인 경우가 많습니다. 또한 레드햇에서 제공하는 부팅 이미지는 새 템플릿이 출시되면 자동으로 업데이트됩니다.

이 섹션에서는 가상머신에서 사용하는 PVC와 함께 오퍼레이터가 생성한 PVC를 살펴보겠습니다.

`openshift-virtualization-os-images` 프로젝트에는 사용 가능한 모든 부팅 소스가 포함되어 있으며 오픈시프트 가상화가 설치되면 자동으로 활성화됩니다.

<br>

1. **스토리지(Storage)** → **PertantVolumeClaim**으로 이동합니다.

   <img src="lab-images/storage_mgmt--2.1_Left_Menu.png" title="100px" alt="스토리지 관리 왼쪽 메뉴"></img> <br> 
<br>

2. 프로젝트 드롭다운을 누르고 `기본 프로젝트 표시를 활성화(Show default projects)`한 후 `openshift-virtualization-os-images` 프로젝트를 선택합니다.

   <img src="lab-images/storage_mgmt--2.2_Select_Project.png" title="100px" alt="프로젝트 선택"></img> <br> 
<br>

3. 오픈시프트 가상화가 자동으로 생성한 부팅 소스를 나열합니다.

   <img src="lab-images/storage_mgmt--2.3_List_PVCs.png" title="100px" alt="PVC 리스트"></img> <br>
<br>

4. 자세한 정보를 얻으려면 목록에서 하나(예: `fedora-XX`)를 선택하십시오.

   이 실습에서 각 PVC는 OpenShift Data Foundation에서 제공하는 `ocs-storagecluster-ceph-rbd` 스토리지 클래스에서 가져옵니다. PVC의 세부정보를 보면 실시간 마이그레이션에 필요하고 `블록(block)` 모드를 사용하는 `ReadWriteMany` PVC임을 알 수 있습니다. 모드는 스토리지 공급업체에 따라 `블록(block)` 또는 `파일(file)`이 될 수 있으며 `RWX` 모드를 사용할 수 있는 한 둘 중 하나가 동일하게 작동합니다.

   용량은 이 템플릿 디스크에서 생성된 모든 가상머신에서 사용할 기본 운영체제와 설치된 모든 패키지/소프트웨어를 수용할 수 있을 만큼 충분히 커야 합니다. 디스크를 복제하여 생성된 가상머신은 디스크 크기를 늘릴 수 있지만 줄일 수는 없습니다.

   <img src="lab-images/storage_mgmt--2.4_PVC_Info.png" title="100px" alt="PVC 정보"></img> <br>
<br>

5. 가상머신이 생성되면 부팅 소스 이미지가 복제되고 새 디스크가 생성됩니다. `vmexamples` 프로젝트로 전환하고 PVC(디스크) 목록을 검토합니다.

   이 프로젝트의 가상머신 용 각 디스크에 대한 PVC와 이 워크숍의 이전 섹션에서 생성된 Microsoft Windows Server 2019 디스크 이미지용 PVC가 표시됩니다. 원하는 경우 마이크로소프트 윈도우 ISO와 함께 이 PVC를 다른 가상 머신에서 재사용하여 운영 체제를 설치할 수 있습니다.

   <img src="lab-images/storage_mgmt--2.5_List_PVCs_VMs.png" title="100px" alt="PVC를 사용하는 가상머신 리스트"></img> <br>
<br>

6. 정보를 얻으려면 `fedora02`를 선택하십시오.

   <img src="lab-images/storage_mgmt--2.6_PVC_VM_Info.png" title="100px" alt="PVC 상세 정보"></img> <br>
<br>

7. *Persistent Volume Claim*은 볼륨을 프로비저닝하기 위해 특정 *스토리지 클래스(Storage class)* 또는 기본 스토리지 클래스를 신청합니다. 목록을 얻으려면 **Storage** → **PertantVolumes**로 이동하십시오. **Claim** 기준으로 정렬합니다.

   <img src="lab-images/storage_mgmt--2.7_PV_List.png" title="100px" alt="PV 리스트"></img> <br>
<br>

8. 이제 **가상화(Virtualization)** → **부팅 가능한 볼륨(Bootable volumes)** 으로 이동하여 사용 가능한 볼륨 목록을 얻습니다.

   <img src="lab-images/storage_mgmt--2.8_List_Bootable_Volumes.png" title="100px" alt="부팅 가능한 볼륨 리스트"></img> <br>
<br>
<br>

## 3. 스냅샷

### 3.1 스냅샷 소개

오픈시프트 가상화는 CSI 스토리지 공급자의 스냅샷 기능을 사용하여 가상머신에 대한 디스크 스냅샷을 생성합니다. 이 스냅샷은 가상머신이 실행 중인 동안 "온라인"으로, 가상머신의 전원이 꺼진 동안 "오프라인"으로 생성될 수 있습니다. KVM 통합 도구가 가상머신에 설치된 경우 게스트 운영체제를 정지(quiescing)할 수 있는 옵션도 제공됩니다. (quiescing은 디스크의 스냅샷이 게스트 파일 시스템의 일관된 상태를 나타내도록 보장합니다. 예를 들어 버퍼는 플러시되고 저널은 일괄됩니다)

디스크 스냅샷은 CSI에 의해 추상화된 스토리지 구현에 따라 달라지므로 성능에 미치는 영향과 사용되는 용량은 스토리지 제공업체에 따라 달라집니다. 스토리지 공급업체와 협력하여 시스템이 PVC 스냅샷을 관리하는 방법과 그 영향이 미칠 수 있거나 없을 수 있는 영향을 결정하십시오.

> [!IMPORTANT]
> 스냅샷 자체는 백업이나 재해 복구 기능이 아닙니다. 스토리지 시스템 장애를 복구하려면 하나 이상의 복사본을 다른 위치에 저장하는 등 다른 방법으로 데이터를 보호해야 합니다. <br>
> <br>
> OADP(OpenShift API for Data Protection) 외에도 Kasten by Veeam, Trilio, Storware와 같은 파트너는 필요에 따라 가상머신을 동일한 클러스터 또는 다른 클러스터에 백업하고 복원하는 기능을 지원합니다.

가상머신 스냅샷 기능을 사용하여 클러스터 관리자와 애플리케이션 개발자는 다음을 수행할 수 있습니다.
* 새 스냅샷 만들기
* 특정 가상머신에 연결된 모든 스냅샷 나열
* 가상머신을 스냅샷으로 되돌리기
* 기존 가상머신 스냅샷 삭제
<br>

### 3.2 스냅샷으로 작업

1. **Virtualization** → **VirtualMachines**으로 다시 이동하여 `vmexamples` 프로젝트에서 가상머신 `fedora02`를 선택합니다.

   <img src="lab-images/storage_mgmt--3.2.1_VM_Overview.png" title="100px" alt="fedora02 가상머신 개요"></img> <br>
<br>

2. **스냅샷(Snapshots)** 탭으로 이동합니다.

   <img src="lab-images/storage_mgmt--3.2.2_VM_Snapshots_Tab.png" title="100px" alt="가상머신의 스냅샷 탭"></img> <br>
<br>

3. **스냅샷 찍기(Take snapshot)** 를 누르면 대화상자가 열립니다.

   <img src="lab-images/storage_mgmt--3.3.3_VM_Snapshot_Dialog.png" title="100px" alt="가상머신 스냅샷 대화창"></img> <br>

> [!NOTE]
> `cloudinitdisk`가 스냅샷에 포함되지 않는다는 경고가 표시됩니다. 이는 임시(ephemeral) 디스크이기 때문에 예상되는 현상이며 발생합니다.
<br>

4. **저장(Save)** 을 누르고 *스냅샷(Snapshot)* 이 생성되고 **상태(status)** 가 `성공(Succeeded)`으로 표시될 때까지 기다립니다.

   <img src="lab-images/storage_mgmt--3.3.4_VM_Snapshot_Taken.png" title="100px" alt="가상머신의 스냅샷 확인"></img> <br>
<br>

5. 세 개의 점을 누르고 가상머신이 실행 중이므로 **복원(Restore)** 옵션이 회색으로 표시되는지 확인합니다.

   <img src="lab-images/storage_mgmt--3.3.5_VM_Restore_Disabled.png" title="100px" alt="가상머신 복구 옵션 확인"></img> <br>
<br>

6. **콘솔(Console)** 탭으로 전환하여 실행 중인 가상머신을 수정합니다. 이 작은 수정으로 인해 가상머신이 중단되고 더 이상 부팅할 수 없습니다.

   수정하면 가상머신이 중단되고 더 이상 부팅할 수 없습니다.

   사용자 `fedora`와 비밀번호 `ocpVirtIsGre@t`(또는 이전 모듈에서 사용한 비밀번호)로 로그인합니다. 다음 명령을 실행합니다.
   ```bash
   [fedora@fedora02 ~]$ sudo rm -rf /boot/grub2; sudo shutdown -r now
   ```
<br>

7. *가상머신*을 부팅할 수 없습니다.

   <img src="lab-images/storage_mgmt--3.3.7_VM_Crashed.png" title="100px" alt="가상머신 장애"></img> <br>

> [!IMPORTANT]
> 이전 단계에서는 운영체제가 게스트 내에서 종료되었습니다. 그러나 오픈시프트 가상화는 기본적으로 자동으로 다시 시작합니다. 이 동작은 전체적으로 또는 가상머신 별로 변경할 수 있습니다.
<br>

8. **작업(Actions)** 드롭다운 메뉴를 사용하여 *가상머신*을 중지합니다. 가상머신이 중지될 때까지 기다립니다.

<br>

9. **스냅샷(Snapshot)** 탭으로 다시 이동하여 이전에 생성된 스냅샷에서 **복원(Restore)** 을 누릅니다.

   <img src="lab-images/storage_mgmt--3.3.9_VM_Restore.png" title="100px" alt="가상머신 복구"></img> <br>
<br>

10. 표시된 대화 상자에서 **복원(Restore)** 을 누릅니다.

    <img src="lab-images/storage_mgmt--3.3.10_VM_Restore_Dialog.png" title="100px" alt="가상머신 복구 대화창"></img> <br>
<br>

11. 가상머신이 복원될 때까지 기다린 후 가상머신을 시작합니다.

    <img src="lab-images/storage_mgmt--3.3.11_VM_Restored.png" title="100px" alt="복구된 가상머신"></img> <br>
<br>

12. 가상머신이 다시 올바르게 부팅되는지 확인합니다.
    
    <img src="lab-images/storage_mgmt--3.3.12_VM_Running.png" title="100px" alt="실행 중인 가상머신 확인"></img> <br>
<br>
<br>

## 4. 가상머신 복제

복제는 디스크 이미지를 저장용으로 사용하는 새 가상머신을 생성하지만 대부분의 복제 구성과 저장된 데이터는 원본 가상머신과 동일합니다.

<br>

1. **작업(Actions)** 메뉴에서 **복제(Clone)** 를 누르면 대화 상자가 열립니다.

   <img src="lab-images/storage_mgmt--4.1_VM_Clone_Dialog.png" title="100px" alt="가상머신 복제 대화창"></img> <br>

> [!NOTE]
> 가상머신의 전원이 켜져 있으면 복제를 수행하기 위해 가상머신이 중지됩니다. 가상머신의 스냅샷이 있는 경우 가상머신 전원을 끄지 않고도 스냅샷에서 복제본을 생성할 수도 있습니다.
<br>

2. 새 가상머신이 생성되고 디스크가 복제되며 자동으로 포털이 새 가상머신으로 리디렉션됩니다.

   <img src="lab-images/storage_mgmt--4.2_VM_Cloned.png" title="100px" alt="복제된 가상머신"></img> <br>

> [!IMPORTANT]
> 복제된 가상머신은 원본 가상머신과 동일한 ID를 갖게 되므로 가상머신과 상호 작용하는 애플리케이션 및 다른 클라이언트와 충돌이 발생할 수 있습니다. 외부 네트워크에 연결되어 있거나 동일한 프로젝트에 있는 가상머신을 복제할 때는 주의하세요.
<br>
<br>

## 5. 요약

이 모듈에서는 오픈시프트 스토리지의 기본 개념을 배웠고 오픈시프트 가상화가 일부 게스트 운영 체제에 대한 부팅 소스를 자동으로 다운로드하고 생성하는 방법을 검토했습니다. 또한 스냅샷 생성, 스냅샷 복원, 가상머신 복제 등 가상머신에 대한 스토리지 관련 작업을 수행했습니다.

### 5.1 용어집

**CSI(Container Storage Interface: 컨테이너 스토리지 인터페이스)**
* 다양한 CO(Container Orchestration: 컨테이너 오케스트레이션) 시스템 전반에서 컨테이너 스토리지를 관리하기 위한 API 사양입니다.
* 오픈시프트 클러스터에는 다양한 공급업체의 많은 CSI 프로비저너가 있을 수 있으며, 각 가상머신은 충돌 없이 여러 공급업체의 스토리지를 사용할 수 있습니다.
<br>

**동적 프로비저닝 (Dynamic Provisioning)**
* 스토리지 프레임워크를 사용하면 필요에 따라 볼륨을 생성할 수 있으므로 클러스터 관리자가 영구 스토리지를 사전 프로비저닝할 필요가 없습니다.
* 각 가상머신 디스크는 1:1 비율로 동적으로 생성된 스토리지 볼륨에 저장됩니다.
<br>

**영구 볼륨 (PV:Persistent Volume)**
* 오픈시프트 가상화는 쿠버네티스 영구 볼륨(PV) 프레임워크를 사용하여 클러스터 관리자가 클러스터에 영구 스토리지를 프로비저닝할 수 있도록 합니다.
* 가상머신은 기본 스토리지 인프라에 대한 구체적인 지식 없이도 PVC를 사용하여 PV 리소스를 요청합니다.
<br>

**영구 볼륨 클레임 (PVCs: Persistent Volume Claims)**
* PVC는 스토리지 용량에 대한 요청이며, PV에 바인딩될 때 시스템이 가상머신에 마운트할 스토리지 볼륨을 아는 방법입니다.
* 가상머신 사용자는 기본 인프라 환경에 대한 세부 정보를 몰라도 스토리지를 사용할 수 있습니다.
<br>

**스토리지 클래스 (Storage Class)**
* *스토리지 클래스(Storage Class)* 는 관리자가 제공하는 스토리지의 클래스(예: "골드", "실버" 및 "브론즈")를 설명하는 방법을 제공합니다.
* 다양한 클래스는 클러스터 관리자가 결정한 서비스 수준, 백업 정책 및 임의 정책의 품질에 매핑될 수 있습니다.
* 이는 스토리지 공급업체에 따라 다릅니다.
<br>
<br>

------
[차례](../README.md) &nbsp;&nbsp;&nbsp;&nbsp; [<< 네트워크 관리 <<](./network_management.md) &nbsp;&nbsp;&nbsp;&nbsp; [>> 가상머신 마이그레이션 >>](./vm_migration.md)