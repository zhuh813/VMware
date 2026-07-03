# VMware
VMware架構圖
graph TB
    %% 1. 全域精緻色系樣式定義
    classDef physical fill:#eceff1,stroke:#37474f,stroke-width:2px;
    classDef workstation fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef pool fill:#fff3e0,stroke:#f57c00,stroke-width:1.5px;
    classDef vm fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef vss fill:#ffebee,stroke:#c62828,stroke-width:1px;
    classDef net fill:#e0f2f1,stroke:#004d40,stroke-width:1px;
    classDef storage fill:#f3e5f5,stroke:#6a1b9a,stroke-width:1.5px;

    %% ==========================================
    %% 頂層：Client Network & Management 服務與管理控制層
    %% ==========================================
    subgraph TopLayer ["【頂層】外部網域與中央管理層"]
        direction LR
        CN["客戶網域 (Client Network)<br>外部/客端網路連結"]
        VM_VC["vCenter Server (VC)<br>192.168.216.250<br>• 集中式中央管理平台"]
        VM_AD["ad2016 (AD / DNS)<br>192.168.216.100<br>• 網域主控與網域名稱解析"]
        class VM_VC,VM_AD vm;
    end

    %% ==========================================
    %% 中層：Compute Nodes & Virtual Network 虛擬化運算與網路技術層
    %% ==========================================
    subgraph ClusterLayer ["【中層】vSphere 運算叢集與虛擬交換器技術 (VMnet 11 Host-only)"]
        direction TB

        %% ESXi 01 詳細架構
        subgraph ESXI1 ["ESXi 01 主機 (192.168.216.101)"]
            direction TB
            VM01["VM01 虛擬機器<br>(CentOS 巢狀虛擬化)"]
            Datastore[(Local Datastore)]
            class VM01 vm;
            class Datastore storage;

            %% ESXi 01 內部的 VSS 交換器群
            subgraph VSS0 ["VSS 0 (Management)"]
                MGMT["mgmt 管理網路"]
            end
            subgraph VSS1 ["VSS 1 (Production)"]
                PG["Port Group"]
                NICTeam["NIC Teaming 負載平衡"]
                PG --- NICTeam
            end
            subgraph VSS2 ["VSS 2 (Storage)"]
                iSCSI["iSCSI 流量網卡"]
            end
            subgraph VSS3 ["VSS 3 (vMotion)"]
                vMotion01["vMotion 遷移介面"]
            end
            class VSS0,VSS1,VSS2,VSS3 vss;
        end

        %% ESXi 02 簡化架構 (對應叢集另一端)
        subgraph ESXI2 ["ESXi 02 主機 (192.168.216.102)"]
            direction TB
            VM01_Target["[ 遷移目標 ]<br>VM01 (接管運算)"]
            vMotion02["vMotion 遷移介面"]
            class VM01_Target vm;
        end
    end

    %% ==========================================
    %% 底層：Physical & Resource Pool 實體電腦與基礎資源池層
    %% ==========================================
    subgraph InfraLayer ["【底層】Win 11 實體主機與 Workstation 虛擬化資源池"]
        direction LR
        ShareStorage[(StarWind 共享儲存硬碟<br>iSCSI SAN)]
        class ShareStorage storage;

        subgraph Hypervisor ["Hypervisor 資源池 (Intel-VT 巢狀技術)"]
            direction TB
            CompPool["Computation Pool<br>(CPU / GPU / RAM)"]
            NetPool["Network Pool<br>(虛擬交換器與實體網卡)"]
            StorePool["Storage Pool<br>(本地硬碟資源)"]
            class CompPool,NetPool,StorePool pool;
        end
    end

    %% ==========================================
    %% 連線關係與邏輯流向對應 (與 vmic 串接)
    %% ==========================================
    
    %% 1. 底層資源池提供上層 vmic
    CompPool --- vmic0 & vmic1 & vmic2 & vmic3 & vmic4 & vmic5
    
    %% 2. ESXi01 的 vmic 對應到內部的 VSS
    vmic0 --- VSS0
    vmic1 --- NICTeam
    vmic2 --- NICTeam
    vmic3 --- NICTeam
    vmic4 --- VSS2
    vmic5 --- VSS3

    %% 3. ESXi02 與 vCenter 的資源簡化串接
    NetPool --- vmic_esxi2["vmic..."] ---> ESXI2
    NetPool --- vmic_vc["vmic..."] ---> VM_VC

    %% 4. 共享儲存與服務邏輯
    ShareStorage <-->|iSCSI Target 提供| VM_AD
    iSCSI <==>|掛載共享儲存 LUN| ShareStorage
    ESXI2 <==>|共同掛載共享儲存| ShareStorage

    %% 5. 核心實驗動態：vMotion 線上移轉流向
    vMotion01 ===>|vMotion 私有網路遷移記憶體狀態| vMotion02
    VM01 -.->|熱移轉| VM01_Target

    %% 6. 中央集中管理與對外連線
    VM_VC -.->|控制與調度| ESXI1
    VM_VC -.->|控制與調度| ESXI2
    
    VM_AD --> CN
    VM01 --> CN
    ESXI2 --> CN
    VM_VC --> CN

    %% 套用群組樣式
    class TopLayer,ClusterLayer,ESXI1,ESXI2 net;
    class InfraLayer physical;
    class Hypervisor workstation;
