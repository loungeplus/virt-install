# 仮想マシンのネットワーク管理

## はじめに

2章で述べたように、すべての仮想マシンはデフォルトで OpenShiftの提供するソフトウェア定義ネットワーク（SDN）に接続されています。
このSDNを *Primary Network*と呼びます。

Primary Networkにより、展開した仮想マシンは、OpenShift クラスター上の他の仮想マシンやコンテナからのアクセスが可能になり、仮想マシンと仮想マシン上に起動するアプリケーションを、より近代的な方法で管理できるようになります。

* SDN は、クラスタ内で VM または Pod としてデプロイされたアプリケーションを、抽象化、接続、公開するための追加機能を提供します。これには、OpenShift の *Service* および *Route* 機能が含まれます。
* OpenShift のネットワークポリシーエンジンにより、VM のユーザーまたは管理者は、個々の VM または *Project* / *Namespace* 全体に対するネットワークトラフィックを許可または拒否するルールを作成することができます。

しかし、必要に応じて仮想マシンをタグなしネットワークや VLAN などの 1 つ以上の物理ネットワークに直接接続したいユースケースもあるでしょう。

OpenShift Virtualizationは、この要件に対し、*Secondary Network*　として、Linux ブリッジなどのホストネットワークの設定によって、仮想マシンをSDNとは異なるネットワークへ接続する機能を提供します。

この機能は、SDN に加えて行われるもので、例えば、管理者は外部 IP アドレスから VM に接続でき、VM はレイヤー2 ネットワークを使用して直接接続できます。

本トピックでは、VM をブリッジ接続し、物理ネットワークに直接接続できるようにするための *Network Attachment Definition* を作成する手順を説明します。

## Kubernetes NMState Operatorのインストール
仮想マシンを Secondary Networkへ接続するには、*Kubernetes NMState Operator*が必要です。

*Kubernetes NMState Operator* は、NMState を使用して OpenShift Container Platform クラスタのノード全体でステート駆動型のネットワーク構成を実行するための Kubernetes API を提供します。 

Kubernetes NMState Operator は、クラスタノード上のさまざまなネットワークインターフェースタイプ、DNS、ルーティングを構成するための機能を提供します。さらに、クラスターノード上のデーモンが、各ノードのネットワークインターフェースの状態を定期的にAPIサーバーに報告します。

本ハンズオン環境には、NMState Operatorがインストールされています。そのため、まずは、NMState Operatorをインストールすることから始めましょう。

`[管理者向け表示]` > `[Operator]` > `[OperatorHub]`を開き、検索ボックスへ `NMState`と入力します。

![](images/3-networking/install-nmstate-operator-1.png)

そして、`[インストール]`ボタン を続けて押下し、NMState Operatorをインストールします。

![alt text](images/3-networking/install-nmstate-operator-2.png)


インストールボタンを押下後、`[インストール済みのOperator]`画面で、`Kubernetes NMState Operator`の状態が`Succeeded`になるまで待ちましょう。

![alt text](images/3-networking/install-nmstate-operator-3.png)

状態が`Succeeded`になったら、`Kubernetes NMState Operator`を開き、`[NMState]`の`インスタンスの作成`をクリックします。

![alt text](images/3-networking/install-nmstate-operator-4.png)

そして、デフォルトの設定のまま`[作成]`ボタンを押下してください。

`[ホーム]` > `[概要]`の`クラスタインベントリ`にて、作成中のPodが存在しなくなるまで待ちましょう。

![alt text](images/3-networking/install-nmstate-operator-5.png)


左側のメニューで `[Virtualzation]` > `Networking` をクリックし、次に `NodeNetworkState` > `[Expand all]` をクリックして現在のノード上のネットワーク構成を確認します。

![alt text](images/3-networking/01_NodeNetworkState_List.png)

現状は、Primary Networkを提供するための デフォルトの OVSブリッジとインタフェースのみが存在します。

## NodeNetworkConfigurationPolicyの作成

ノード上に新たなネットワーク構成を追加するには、`NodeNetworkConfigurationPolicy`を作成します。

### ノードラベルの追加

`NodeNetworkConfigurationPolicy`を適用するノードは、ラベルを元に選択されます。
1章で追加したベアメタルインスタンスのみに、`NodeNetworkConfigurationPolicy`を適用するために、ベアメタルインスタンスを識別するためのラベルを追加しておきましょう。

`[管理者向け表示]` > `[コンピュート]` > `[ノード]`を開き、インスタンスタイプが`c5n.metal`のノードの メニューをクリックして、`[ラベルの編集]`を開きます。
![alt text](image-3.png)

`node-type=baremtal`というラベルを入力して、`[Enter]`した後、 `[保存]`ボタンを押下してください。

![alt text](image-4.png)

> Note. 1章にて、c5n.metalのノードを2台追加しているため、もう一台に対しても同様の手順でラベルを追加してください。


### NodeNetworkConfigurationPolicyの作成

![alt text](image-5.png)

![alt text](image-6.png)



```
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: create-br0-static
spec:
  nodeSelector:
    node-type: baremetal 
  desiredState:
    interfaces:
      - name: br0
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
          address:
            - ip: 192.168.95.1
              prefix-length: 24
          dhcp: true
        bridge:
          options:
            stp:
              enabled: false
```

![alt text](image-7.png)


![alt text](image-8.png)


## Network Attachment Definitionの作成

VMでLinuxブリッジを使用するには、*Network Attachment Definition* を作成する必要があります。これは、OpenShiftにネットワークを通知し、仮想マシンがネットワークに接続できるようにするものです。Network Attachment Definitionはプロジェクトに紐づいており、そのプロジェクトにデプロイされた仮想マシンだけがアクセスできます。 Network Attachment Definitionがデフォルトのプロジェクトに作成された場合、グローバルに利用可能になります。 これにより、管理者は、VMを管理するアクセス権を持つ特定のユーザーに対して、どのネットワークを利用可能にするか、または利用不可能にするかを制御することができます。

> NOTE: Network Attachment Definitionは、既存のネットワークデバイスを利用するようにOpenShiftに指示します。この例では、そのデバイスは以前に作成されており、*br-flat* という名前が付けられています。この名前を使用する必要があります。OpenShiftは、その名前のネットワークデバイスが接続されているノードのみを利用できるため、VMを任意のコンピュートノードに配置できなくなります。

左側のメニューから *Network*、*Network Attachment Definition* の順に選択し、*Create Network Attachment Definition* ボタンをクリックします。

![alt text](images/3-networking/06_NetworkAttachDefinition_Create.png)


> IMPORTANT: Network Attachment Definitionを作成する際には、vmexamples-{user}プロジェクト内であることを確認してください。

*vmexamples-{user}* プロジェクト用のフォームを以下のように入力し、*Create Network Attachment Definition* をクリックします。
* *Name*: flatnetwork
* *Description*: CNV Linux Bridge
* *Network Type*: Linux Bridge
* *Bridge name*: br-flat

![alt text](images/3-networking/07_NetworkAttachDefinition_Create_Form.png)


> NOTE: 上記のフォームには、VLAN タグ番号を入力するフィールドがあります。これは、VLAN タグの割り当てが必要なネットワークに接続する場合に使用します。このラボでは、タグなしネットワークを使用しているため、VLAN 番号は必要ありません。

> NOTE: ホスト上の単一のLinuxブリッジには、多くの異なるVLANを関連付けることができます。このシナリオでは、個々のNetwork Attachment Definitionを作成するだけでよく、個別のホストインターフェースやブリッジを作成する必要はありません。

Network Attachment Definitionの詳細を確認します。これは *vmexamples-{user}* プロジェクトで作成されたため、他のプロジェクトでは利用できません。

![alt text](images/3-networking/08_NetworkAttachDefinition_Created.png)

## 仮想マシンをネットワークに接続
左側のメニューで *VirtualMachines* に移動し、中央の列から *fedora01* VM を選択します。 *Configuration* タブをクリックし、左側の *Network* タブをクリックします。

![alt text](images/3-networking/09_VM_Network_Tab.png)

*ネットワークインターフェースの追加* をクリックし、表示されるフォームに必要事項を入力して、*保存* をクリックします。


![alt text](images/3-networking/10_VM_Network_Attach.png)


> NOTE: これは外部ネットワークに接続するブリッジであるため、ネットワークを使用する仮想マシン用のマスカレード（NAT）など、アクセスを有効にするためにOpenShiftの機能や能力に頼る必要はありません。そのため、ここでは *Type* は *Bridge* であるべきです。

*アクション* メニューまたは *Start* ボタンを使用してVMを起動し、*コンソール* タブに切り替えて起動を確認します。

![alt text](images/3-networking/11_VM_Network_Startup.png)


*enp2s0* インターフェースは、フラットネットワーク（*192.168.64.0/18*）からIPアドレスを取得します。そのネットワークには、そのネットワークにIPを割り当てるDHCPサーバーがあります。 
![alt text](images/3-networking/12_VM_Network_Console.png)

fedora02 VMを同じ *flatnetwork* ネットワークにアタッチするために、同じ手順を繰り返します。

コンソールで *ping* コマンドを使用して、2つのVM（fedora01とfedora02）間の直接通信を実演します。
![alt text](images/3-networking/13_VM_Network_Ping.png)

## User Defined Network

User Defined Network（UDN）の実装前は、OpenShift Container Platform用のOVN-Kubernetes CNIプラグインはプライマリまたはメインネットワーク上のレイヤー3トポロジーのみをサポートしていました。Kubernetesの設計原則により、すべてのPodはメインネットワークに接続され、すべてのPodはIPアドレスを使用して相互に通信し、Pod間のトラフィックはネットワークポリシーに従って制限されます。新しいネットワークアーキテクチャを学ぶことは、多くの従来の仮想化管理者からしばしば表明される懸念事項です。

UDNの導入により、カスタムのレイヤ2、レイヤ3、ローカルネットのネットワークセグメントが有効になり、KubernetesのPod Networkのデフォルトのレイヤ3トポロジーの柔軟性とセグメント化機能が向上します。これらのセグメントは、デフォルトのOVN-Kubernetes CNIプラグインを使用するコンテナPodや仮想マシンに対して、プライマリまたはセカンダリネットワークとして機能します。UDNは、幅広いネットワークアーキテクチャとトポロジーを可能にし、ネットワークの柔軟性、セキュリティ、およびパフォーマンスを向上させます。

クラスタ管理者は、ClusterUserDefinedNetworkカスタムリソース（CR）を活用することで、UDNを使用してクラスタレベルで複数のネームスペースにまたがる追加のネットワークを作成および定義できます。さらに、クラスタ管理者またはクラスタユーザーは、UserDefinedNetwork CRを使用して、ネームスペースレベルで追加のネットワークを定義するためにUDNを使用できます。

User Defined Networkには、以下の利点があります。

*セキュリティ強化のためのネットワーク分離* - ネームスペースは、テナントが Red Hat OpenStack Platform (RHOSP) で分離されるのと同様に、独自の分離されたプライマリネットワークを持つことができます。これにより、テナント間のトラフィックのリスクが低減され、セキュリティが向上します。

*ネットワークの柔軟性* - クラスター管理者は、プライマリネットワークをレイヤー2またはレイヤー3のネットワークタイプとして構成できます。これにより、プライマリネットワークにセカンダリネットワークの柔軟性が提供されます。

*簡素化されたネットワーク管理* - User Defined Networkにより、異なるネットワークでワークロードをグループ化することで分離が実現できるため、複雑なネットワークポリシーの必要性がなくなります。

*高度な機能* - User Defined Networkにより、管理者は複数のネームスペースを単一のネットワークに接続したり、異なるネームスペースのセットごとに個別のネットワークを作成したりすることができます。 また、ユーザーは異なるネームスペースやクラスターにまたがって IP サブネットを指定し、再利用することもでき、一貫したネットワーク環境を提供します。


### OpenShift VirtualizationによるUser Defined Network

OpenShift Container Platform のウェブコンソールまたは CLI を使用して、仮想マシン（VM）のプライマリインターフェイス上のUser Defined Network（UDN）に仮想マシンを接続することができます。プライマリUser Defined Networkは、指定したネームスペースのデフォルトのPod Networkに置き換わります。Pod Networkとは異なり、プライマリ UDN はプロジェクトごとに定義でき、各プロジェクトは固有のサブネットとトポロジーを使用できます。

レイヤー2トポロジーでは、OVN-Kubernetesはノード間にオーバーレイネットワークを作成します。このオーバーレイネットワークを使用すると、追加の物理ネットワークインフラストラクチャを構成することなく、異なるノード上のVMを接続することができます。

レイヤー2トポロジーでは、ライブマイグレーション時に永続的なIPアドレスがクラスターノード全体で保持されるため、ネットワークアドレス変換（NAT）を必要とせずにVMのシームレスなマイグレーションが可能です。

プライマリUDNを実装する前に、以下の制限事項を考慮する必要があります。

- virtctl ssh コマンドを使用して VM への SSH アクセスを構成することはできません。
- oc port-forward コマンドを使用して VM へのポート転送を行うことはできません。
- Headlessサービスを使用して VM にアクセスすることはできません。
- VM の健全性チェックを構成するためのReadinessおよびLivenessのプローブを定義することはできません。

> NOTE: OpenShift Virtualization は現在、セカンダリUser Defined Networkをサポートしていません。

### User Defined Networkの使用

UDNにアクセスできるPodを作成する前に、ネームスペースとネットワークを作成する必要があります。Podを新しいネットワークにネームスペースを割り当てることや、既存のネームスペースにUDNを作成することは、OVN-Kubernetesでは受け付けられません。

この作業はクラスタ管理者によって実行する必要があります。*vmexamples-{user}-udn* という名前空間を適切なラベル（*k8s.ovn.org/primary-user-defined-network*）とともに割り当てました

*Network* に移動し、*User Defined Network* をクリックして、プロジェクト *vmexamples-{user}-udn* が選択されていることを確認する。

![alt text](images/3-networking/14_UDN_List.png)

*Create* をクリックし、 *UserDefinedNetwork* を選択します。
![alt text](images/3-networking/15_UDN_Create.png)

サブネット *192.168.254.0/24* を指定し、 *Create* をクリックします。
![alt text](images/3-networking/16_UDN_Form.png)

作成したUDNの設定を確認します。
![alt text](images/3-networking/17_UDN_Created.png)

* フォームを使用して作成した場合のデフォルト名は *primary-udn* です。
* デフォルトではレイヤー2です（現時点でOpenShift仮想化でサポートされている唯一のレイヤー）。
* 役割はプライマリです（仮想マシンは現時点ではプライマリネットワークのみを使用できます）。
* Network Attachment Definitionは自動的に作成されます。

次に、左側のメニューで *NetworkAttachmentDefinitions* に移動し、関連するNADが自動的に作成されていることを確認します。
![alt text](images/3-networking/18_UDN_NAD.png)

UserDefinedNetworkに接続された仮想マシンを作成するには、 https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/virtualization/networking#virt-connecting-vm-to-primary-udn[YAML定義の調整^] が必要です。このラボでは作業を簡単にするため、以下のYAML定義を使用し、UserDefinedNetworkに接続されたVMを作成します。

次の画像のように、トップメニューを使用してYAMLをインポートできます。
![alt text](images/3-networking/19_UDN_Import_YAML.png)

```
----
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: fedora-udn
  name: fedora-udn
  namespace: vmexamples-{user}-udn
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        creationTimestamp: null
        name: fedora-udn
      spec:
        sourceRef:
          kind: DataSource
          name: fedora
          namespace: openshift-virtualization-os-images
        storage:
          resources:
            requests:
              storage: 30Gi
  runStrategy: Always
  template:
    metadata:
      name: fedora-udn
      namespace: vmexamples-{user}-udn
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: rootdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
          interfaces:
          - name: primary-udn
            binding:
              name: l2bridge
          rng: {}
        resources:
          requests:
            memory: 2048M
      networks:
      - pod: {}
        name: primary-udn
      terminationGracePeriodSeconds: 0
      volumes:
      - dataVolume:
          name: fedora-udn
        name: rootdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            user: fedora
            password: fedora
            chpasswd: { expire: False }
        name: cloudinitdisk
----
```

貼り付けが完了したら、画面下部の青い *Create* ボタンをクリックしてVMの作成プロセスを開始します。
![alt text](images/3-networking/20_Create_VM_YAML.png)

*VirtualMachines* に切り替えて、VM が作成されるのを見ます。 作成されたら、新たに作成された *fedora-udn* 仮想マシンを確認します。 *Overview* タブの *Network* タイルに、UserDefinedNetwork から割り当てられた IP が表示されます。
![alt text](images/3-networking/21_UDN_Network_Tile.png)

コンソールタブに切り替えて、提供されたゲスト認証情報を使用してVMにログインします。 
![alt text](images/3-networking/22_UDN_Fedora_Console.png)

- VMは定義されたサブネットからIPを割り当てます。
- VMはDHCPからゲートウェイ構成を自動的に取得します。
- VMはUser Defined Networkを使用してインターネットにアクセスできます。

## まとめ

このモジュールでは、物理ネットワークの操作と、仮想マシン（VM）を既存のネットワークに直接接続する方法について学習しました。仮想マシンを物理ネットワークに直接接続することで、管理者は仮想マシンに直接アクセスできるだけでなく、仮想マシンをストレージネットワークや管理ネットワークなどの専用ネットワークに接続することも可能になります。

User Defined Networkは、クラスタ管理者やエンドユーザーに高度なカスタマイズが可能なネットワーク構成オプションを提供し、プライマリおよびセカンダリネットワークの両方をより柔軟に管理することができます。