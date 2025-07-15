= テンプレートとインスタンスタイプの管理

== はじめに

事前設定済みの Red Hat 仮想マシンテンプレートは、*Virtualization*  ページ以下の *Templates* にリストされています。これらのテンプレートは、Red Hat Enterprise Linux、Fedora、CentOS、Microsoft Windows Desktop、および Microsoft Windows Server の異なるバージョンで利用可能です。各 Red Hat 関連の仮想マシンテンプレートは、オペレーティングシステムのイメージ（起動ソース）、オペレーティングシステムのデフォルト設定、フレーバー（CPUとメモリ）、およびワークロードタイプ（サーバー）が事前に設定されています。その他のオペレーティングシステム用のテンプレートには OS イメージは含まれていませんが、そのオペレーティングシステム用に推奨される設定が事前に設定されています。

*Templates* ページには、以下の4種類の仮想マシンテンプレートが表示されます:

* *Red Hat Supported* のテンプレートは、Red Hat により完全にサポートされています。
* *User Supported* のテンプレートは、Red Hat サポート対象のテンプレートをユーザーが複製して作成したものです。
* *Red Hat Provided* のテンプレートは、Red Hat によるサポートが限定的です。
* *User Provided* のテンプレートは、 *Red Hat Provided* テンプレートをユーザーが複製して作成したものです。

[[prepare_templates_lab]]
== ラボの準備

. これから実行する作業では、いくつかの追加のVMをプロビジョニングする必要があります。準備として、共有環境がラボを完了するのに十分なリソースを確保できるよう、既存の *fedora01* および *fedora02* 仮想マシンをシャットダウンしてください。

. 左側のメニューで *Virtualization* パースペクティブに移動し、*Virtualmachines* をクリックします。
. VMワークロードをホストしている、アクセス可能な各プロジェクトが中央列のツリービューに表示されます。（少なくとも、プロジェクト *vmimported-{user}* と *vmexamples-{user}* を展開して、仮想マシンのステータスを確認してください。
. VMのステータスが *Running* と表示されている場合は、中央のツリー列でVMをハイライトし、 *Actions* ドロップダウンメニューから *Stop* ボタンまたはオプションを選択します。

これで、すべてのVMが *Stopped* 状態になります。

image::2025_spring/module-07-tempinst/00_VMs_Stopped.png[link=self, window=blank, width=100%]

[[clone_customize_template]]
== テンプレートの複製とカスタマイズ

デフォルトでは、Red Hat OpenShift Virtualization が提供する事前構成済みのテンプレートはカスタマイズできません。ただし、テンプレートを複製して、特定のワークロードに合わせて調整し、特定のワークロード用の特定のタイプの仮想マシンを簡単に要求できるようにすることは可能です。このラボのこのセクションでは、まさにこの作業を行います。エンドユーザーにオンデマンドで事前構成済みのデータベースサーバーを提供するテンプレートを作成します。

. まず、左側のメニューで *Templates* をクリックし、プロジェクトとして *openshift* を選択します。*openshift* プロジェクトを表示するには、*Show default projects* ボタンを切り替える必要があるかもしれません。
+
image::2025_spring/module-07-tempinst/01_Project_Toggle.png[link=self, window=blank, width=100%]
+
image::2025_spring/module-07-tempinst/01_Template_List.png[link=self, window=blank, width=100%]

. 検索バーに *centos9* と入力し、Enterキーを押します。表示されるテンプレートリストから、*centos-stream9-server-small* のテンプレートを見つけます。
+
image::2025_spring/module-07-tempinst/02_Search_Centos9.png[link=self, window=blank, width=100%]

. *centos-stream9-server-small* のテンプレート名をクリックすると、デフォルトのテンプレートは編集できない旨のメッセージが表示され、クローンを作成するか尋ねられます。*Create a new custom Template* オプションをクリックします。
+
image::2025_spring/module-07-tempinst/03_Create_Custom_Template.png[link=self, window=blank, width=100%]

. *Clone template* という新しいメニューが表示されます。以下の値を入力し、完了したら *Clone* ボタンをクリックします。
+
* *Template name:* centos-stream9-server-db-small
* *Template project:* vmexamples-{user}
* *Template display name:* CentOS Stream 9 VM - Database Template Small
* *Template provider:* Roadshow {user}
+
image::2025_spring/module-07-tempinst/04_Clone_Template_Options.png[link=self, window=blank, width=100%\]

. これにより、テンプレートの *Details* ページに移動し、いくつかのオプションをカスタマイズできるようになります。まず、ページの下部付近にあるCPUとメモリを見つけ、鉛筆アイコンをクリックして編集します。
+
image::2025_spring/module-07-tempinst/05_Clone_Details.png[link=self, window=blank, width=100%\]

. 新しいウィンドウが開き、CPUとメモリの量を編集できます。カスタムテンプレートでは、CPUの値を2、メモリの値を4 GiBに設定し、*Save* ボタンをクリックします。
+
image::2025_spring/module-07-tempinst/06_Edit_CPU_Mem.png[link=self, window=blank, width=100%]

. 次に、画面上部の *Scripts* タブをクリックし、 *Cloud-init* セクションで *Edit* ボタンをクリックします。
+
image::2025_spring/module-07-tempinst/09_Scripts_CloudInit.png[link=self, window=blank, width=100%]

.  *Cloud-init* ダイアログが開いたら、*Configure via: Script* のラジオボタンをクリックし、以下の YAML スニペットで YAML を置き換えます。
+
[source,yaml,role=execute]
----
userData: |-
  #cloud-config
  user: centos
  password: ${CLOUD_USER_PASSWORD}
  chpasswd: { expire: False }
  packages:
    - mariadb-server
  runcmd:
    - systemctl enable mariadb
    - systemctl start mariadb
----
+
image::2025_spring/module-07-tempinst/10_Cloud_Init_Script.png[link=self, window=blank, width=100%]

.  *Save* ボタンをクリックすると、*Saved* という緑色のプロンプトが表示されます。次に、*Apply* ボタンをクリックします。

. 次に、左側のメニューにある *Catalog* 項目をクリックし、 *Template catalog* オプションを選択し、さらに *User templates* を選択します。作成したテンプレートがタイルとして利用可能になっているはずです。
+
image::2025_spring/module-07-tempinst/11_User_Templates.png[link=self, window=blank, width=100%]

.  タイルをクリックすると、VMの起動画面が表示されます。 *Quick create VirtualMachine*（仮想マシンのクイック作成）ボタンをクリックします。
+
image::2025_spring/module-07-tempinst/12_Quick_Create_Template.png[link=self, window=blank, width=100%]

. 仮想マシンが起動すると、*Overview* ページに、テンプレートから作成され、定義した追加リソースが含まれていることが表示されます。あとは、*MariaDB* がインストールされていることを確認するだけです。
+
image::2025_spring/module-07-tempinst/13_VM_From_Template.png[link=self, window=blank, width=100%]

. 上部にある *Console* タブをクリックし、提供された *Guest login credentials* と *Copy* および *Paste to console* ボタンを使用して、仮想マシンのコンソールにログインします。
+
image::2025_spring/module-07-tempinst/14_VM_Console.png[link=self, window=blank, width=100%]

. 仮想マシンにログインしたら、次のコマンドを実行してMariaDBのインストールをテストします。
+
[source,sh,role=execute]
----
sudo mysql -u root
----
+
image::2025_spring/module-07-tempinst/15_MariaDB_Login.png[link=self, window=blank, width=100%]

. VMからログアウトするには、*Ctrl-D* を2回押します。

[[create_win]]
== Windows VMテンプレートの作成

このラボのセグメントでは、WebサーバーにホストされているISOを使用してMicrosoft Windows Server 2019をインストールします。これは、Webサーバー、オブジェクトストレージ、またはクラスター内の他の永続ボリュームなど、多くの場所からディスクをソースする機能を活用して仮想マシンにオペレーティングシステムをインストールする1つの方法です。

このプロセスは、sysprep済みの仮想マシンからクローンルートディスクを作成し、他のテンプレートで使用することで、オペレーティングシステムの初期インストール後に簡素化することができます。

NOTE: テンプレートとして使用するゲストオペレーティングシステムの準備プロセスは、状況によって異なります。テンプレートOSの準備の際には、必ず組織のガイドラインと要件に従ってください。

. 左側のメニューから *Catalog* に移動し、上部の *Template catalog* タブをクリックします。

. 検索バーに *win* と入力するか、または *Microsoft Windows Server 2019 VM* のタイルが見つかるまで下にスクロールします。
+
image::2025_spring/module-07-tempinst/16_Windows_2k19_Tile.png[link=self, window=blank, width=100%]

. テンプレートに関連するデフォルト構成を示すダイアログが表示されます。
+
NOTE: ブートソースが提供されていないため、このVMを素早く作成するオプションが初期状態では表示されないことに注意してください。VMをニーズに合わせてカスタマイズする必要があります。
+
image::2025_spring/module-07-tempinst/17_Windows_2k19_Dialog.png[link=self, window=blank, width=100%]
+
. ダイアログで以下を入力します：
* *win-sysprep* という名前を指定します。
* *Boot from CD* のチェックボックスをオンにします。
* ドロップダウンメニューから *(creates PVC)* URLを選択します。
* *image URL* を指定します : https://catalog-item-assets.s3.us-east-2.amazonaws.com/qcow_images/Windows2019.iso
* CDディスクのサイズを *5 GiB* に縮小します。
* *Disk source* は *Blank* のままにし、サイズはデフォルト値の *60 GiB* に設定します
* *Mount Windows drivers dis* チェックボックスが有効になっていることを確認します。 **これは、VirtIO用のドライバを提供するWindowsシステムをインストールするために必要です。**
+

. オプションを入力したら、テンプレートの設定を続けるために、下部の *Customize VirtualMachine* ボタンをクリックします。
+
image::2025_spring/module-07-tempinst/18_Windows_2k19_Parameters.png[link=self, window=blank, width=100%]

. *Customize and create VirtualMachine* 画面で、*Boot mode* オプションの横にある編集用鉛筆アイコンをクリックします。 
+
image::2025_spring/module-07-tempinst/19_Boot_Mode.png[link=self, window=blank, width=100%]

. *Boot mode* メニューが表示されたら、ドロップダウンメニューから *BIOS* ブートモードを選択し、 *Save* ボタンをクリックします。
+
image::2025_spring/module-07-tempinst/19a_Boot_BIOS.png[link=self, window=blank, width=100%]

. 次に、 *Scripts* タブをクリックし、 *Sysprep* セクションまでスクロールダウンして、 *Edit* ボタンをクリックします。
+
image::2025_spring/module-07-tempinst/20_Customize_Scripts.png[link=self, window=blank, width=100%]

. 新しいウィンドウがポップアップし、新しいテンプレート用の *Sysprep* アクションを作成できます。
+
image::2025_spring/module-07-tempinst/21_Sysprep.png[link=self, window=blank, width=100%]

. 次のコードブロックを *autounattend.xml* セクションにコピーして貼り付けます。
+
[source,xml,role=execute]
----
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="urn:schemas-microsoft-com:unattend">
  <settings pass="windowsPE">
    <component name="Microsoft-Windows-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <DiskConfiguration>
        <Disk wcm:action="add">
          <CreatePartitions>
            <CreatePartition wcm:action="add">
              <Order>1</Order>
              <Extend>true</Extend>
              <Type>Primary</Type>
            </CreatePartition>
          </CreatePartitions>
          <ModifyPartitions>
            <ModifyPartition wcm:action="add">
              <Active>true</Active>
              <Format>NTFS</Format>
              <Label>System</Label>
              <Order>1</Order>
              <PartitionID>1</PartitionID>
            </ModifyPartition>
          </ModifyPartitions>
          <DiskID>0</DiskID>
          <WillWipeDisk>true</WillWipeDisk>
        </Disk>
      </DiskConfiguration>
      <ImageInstall>
        <OSImage>
          <InstallFrom>
            <MetaData wcm:action="add">
              <Key>/IMAGE/NAME</Key>
              <Value>Windows Server 2019 SERVERSTANDARD</Value>
            </MetaData>
          </InstallFrom>
          <InstallTo>
            <DiskID>0</DiskID>
            <PartitionID>1</PartitionID>
          </InstallTo>
        </OSImage>
      </ImageInstall>
      <UserData>
        <AcceptEula>true</AcceptEula>
        <FullName>Administrator</FullName>
        <Organization>My Organization</Organization>
      </UserData>
      <EnableFirewall>false</EnableFirewall>
    </component>
    <component name="Microsoft-Windows-International-Core-WinPE" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <SetupUILanguage>
        <UILanguage>en-US</UILanguage>
      </SetupUILanguage>
      <InputLocale>en-US</InputLocale>
      <SystemLocale>en-US</SystemLocale>
      <UILanguage>en-US</UILanguage>
      <UserLocale>en-US</UserLocale>
    </component>
  </settings>
  <settings pass="offlineServicing">
    <component name="Microsoft-Windows-LUA-Settings" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <EnableLUA>false</EnableLUA>
    </component>
  </settings>
  <settings pass="specialize">
    <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <AutoLogon>
        <Password>
          <Value>R3dh4t1!</Value>
          <PlainText>true</PlainText>
        </Password>
        <Enabled>true</Enabled>
        <LogonCount>999</LogonCount>
        <Username>Administrator</Username>
      </AutoLogon>
      <OOBE>
        <HideEULAPage>true</HideEULAPage>
        <HideLocalAccountScreen>true</HideLocalAccountScreen>
        <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
        <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
        <NetworkLocation>Work</NetworkLocation>
        <ProtectYourPC>3</ProtectYourPC>
        <SkipMachineOOBE>true</SkipMachineOOBE>
      </OOBE>
      <UserAccounts>
        <LocalAccounts>
          <LocalAccount wcm:action="add">
            <Description>Local Administrator Account</Description>
            <DisplayName>Administrator</DisplayName>
            <Group>Administrators</Group>
            <Name>Administrator</Name>
          </LocalAccount>
        </LocalAccounts>
      </UserAccounts>
      <TimeZone>Eastern Standard Time</TimeZone>
    </component>
  </settings>
  <settings pass="oobeSystem">
    <component name="Microsoft-Windows-International-Core" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <InputLocale>en-US</InputLocale>
      <SystemLocale>en-US</SystemLocale>
      <UILanguage>en-US</UILanguage>
      <UserLocale>en-US</UserLocale>
    </component>
    <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <AutoLogon>
        <Password>
          <Value>R3dh4t1!</Value>
          <PlainText>true</PlainText>
        </Password>
        <Enabled>true</Enabled>
        <LogonCount>999</LogonCount>
        <Username>Administrator</Username>
      </AutoLogon>
      <OOBE>
        <HideEULAPage>true</HideEULAPage>
        <HideLocalAccountScreen>true</HideLocalAccountScreen>
        <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
        <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
        <NetworkLocation>Work</NetworkLocation>
        <ProtectYourPC>3</ProtectYourPC>
        <SkipMachineOOBE>true</SkipMachineOOBE>
      </OOBE>
      <UserAccounts>
        <LocalAccounts>
          <LocalAccount wcm:action="add">
            <Description>Local Administrator Account</Description>
            <DisplayName>Administrator</DisplayName>
            <Group>Administrators</Group>
            <Name>Administrator</Name>
          </LocalAccount>
        </LocalAccounts>
      </UserAccounts>
      <TimeZone>Eastern Standard Time</TimeZone>
    </component>
  </settings>
</unattend>
----

. コードを貼り付けたら、ダイアログの *Save* ボタンをクリックします。
+
image::2025_spring/module-07-tempinst/22_Windows_2k19_Sysprep.png[link=self, window=blank, width=100%]

. Sysprepが完了したら、画面の下部にある *Create VirtualMachine* ボタンをクリックします。
+
image::2025_spring/module-07-tempinst/23_Create_VirtualMachine.png[link=self, window=blank, width=100%]

. 仮想マシンは、ISOイメージをダウンロードし、設定を行い、インスタンスを起動することで、プロビジョニングプロセスを開始します。
+
image::2025_spring/module-07-tempinst/24_Windows_2k19_Provisioning.png[link=self, window=blank, width=100%]

. このプロセスは、起動 ISO イメージのダウンロードが必要なため、数分かかる場合があります。 *Diagnostics* タブをクリックすると、ダウンロードの進行状況を確認できます。
+
image::2025_spring/module-07-tempinst/25_CD_Import.png[link=self, window=blank, width=100%]

. しばらくすると仮想マシンが起動し、ステータスが *Running* に変わります。 *Console* タブをクリックして、自動応答のインストールプロセスを表示します。
+
image::2025_spring/module-07-tempinst/26_Windows_2k19_Console.png[link=self, window=blank, width=100%]

. VMのインストールプロセスが完了したら（プロビジョニングには3～5分、起動と設定には約10分かかります）、停止ボタンで電源をオフにします。
+
image::2025_spring/module-07-tempinst/27_Stop_Button.png[link=self, window=blank, width=100%]

. マシンをシャットダウンしたら、今後Windowsテンプレートベースのインストールを行う際に毎回カスタマイズプロセスを実行することなく使用できるルートボリュームのクローンを作成します。

. 左側のメニューで *Storage* をクリックし、次に *PersistentVolumeClaims* をクリックすると、*vmexamples-{user}* ネームスペースで利用可能な PVC のリストが表示されます。

. インストールで作成された *win-sysprep* PVC を見つけ、右側の3点メニューから *Clone PVC* を選択します。
+
image::2025_spring/module-07-tempinst/28_Storage_PVC.png[link=self, window=blank, width=100%]

. ポップアップメニューで以下のオプションを入力し、*Clone*（クローン）ボタンをクリックします。
* *Name*: windows-2k19-sysprep-template
* *Access mode*:  Shared access (RWX) 
* *StorageClass*: ocs-external-storagecluster-ceph-rbd-immediate 
+
image::2025_spring/module-07-tempinst/29_Clone_Menu.png[link=self, window=blank, width=100%]

. これを保存すると、今後Windows VMを素早く作成する際に使用できます。

.  *Catalog* メニュー項目に戻り、*Disk source* として *PVC (clone PVC)* オプションを選択し、*PVC name* として *Windows-2k19-Sysprep-Template* PVCを選択して、クローンを作成します。*Customize VirtualMachine* ボタンをクリックして、ブートモードを *UEFI* ではなく *BIOS* に設定します。
+
image::2025_spring/module-07-tempinst/30_Windows_Template.png[link=self, window=blank, width=100%]

. BIOSを設定し、*Create VirtualMachine*（仮想マシンの作成）をクリックします。
+
image::2025_spring/module-07-tempinst/31_Windows_Template_BIOS.png[link=self, window=blank, width=100%]

. しばらくすると、新しい Windows Server 2019 仮想マシンがクローン作成された PVC から起動します。
+
image::2025_spring/module-07-tempinst/32_Windows_Template_Running.png[link=self, window=blank, width=100%]

[[instance_types]]
== インスタンスタイプの紹介

仮想マシンのデプロイプロセスを簡素化するために、OpenShift 4.14 からデフォルトの構成メカニズムが変更され、*インスタンスタイプ* の使用が強調されるようになりました。インスタンスタイプは、新しいVMに適用するリソースと特性を定義できる再利用可能なオブジェクトです。独自のVMをプロビジョニングする際に、OpenShift Virtualizationをインストールすると、カスタムインスタンスタイプを定義したり、さまざまなインスタンスタイプを使用したりできます。これは、一般的なクラウドプロバイダーのセルフサービスカタログを使用する際にユーザーが経験することに非常に似ています。

. このセクションでは、インスタンスタイプを使用してVMをプロビジョニングする方法を説明します。

. まず、左側のメニューで *Catalog* をクリックします。 デフォルトのカタログ項目として *Instance Types* が表示されます。
+
image::2025_spring/module-07-tempinst/33_Left_Menu_Catalog.png[link=self, window=blank, width=100%]

. インスタンスタイプを使用する最初のステップは、起動するボリュームを選択することです。起動ソースを提供するテンプレートと同様に、これらの起動ソースは、InstanceTypeでプロビジョニングされたゲストで使用できます。*openshift-virtualization-os-images* プロジェクトを選択すると、含まれるボリュームを確認できます。または、*Add volume* ボタンを使用して独自のボリュームをアップロードすることもできます。
+
image::2025_spring/module-07-tempinst/34_Volume_Boot.png[link=self, window=blank, width=100%]

. *rhel9* ブートボリュームをクリックして、起動するボリュームタイプとして選択します。 選択すると、イメージ名の左側に小さな青い縦線が表示され、名前自体が太字に変わります。
+
image::2025_spring/module-07-tempinst/35_Select_RHEL9.png[link=self, window=blank, width=100%]

. 次に、使用するインスタンスタイプを選択できます。デフォルトで Red Hat が提供するインスタンスタイプが用意されていますが、独自のインスタンスタイプを作成して特定の用途に使用することもできます。提供されているインスタンスタイプにカーソルを合わせると、その使用目的の説明が表示されます。
+
image::2025_spring/module-07-tempinst/36_Select_InstanceType.png[link=self, window=blank, width=100%]
+
* Red Hat が提供するインスタンスタイプは、以下の用途を想定しています。
** *Nシリーズ*: VNFs のようなネットワーク集約的な DPDK ワークロード用に設計されています。
** *Oシリーズ*：メモリオーバーコミットが事前構成された、特殊な汎用インスタンスタイプです。
** *CXシリーズ*：追加の専用CPUをリクエストすることで、追加の機能オフロードによる計算集約型ワークロード向けに設計されています。
** *Uシリーズ*：最も汎用性の高い、または「ユニバーサル」なインスタンスタイプです。
** *Mシリーズ*：メモリ集約型ワークロード向けに設計されています。

. *Uシリーズ* のタイルをクリックすると、一般的なインスタンスタイプの定義済みリソースのドロップダウンリストが表示されます。 デフォルトのオプションは *medium: 1 CPUs, 4 GiB Memory* です。これを選択します。 選択すると、インスタンスタイプのフォントが青字で太字表示されます。
+
image::2025_spring/module-07-tempinst/37_InstanceType_Resources.png[link=self, window=blank, width=100%]

. インスタンスタイプを使用してプロビジョニングを行う際に最後に完了させる必要があるセクションは、テンプレートセクションと類似しています。仮想マシンに名前を付け、バックアップディスクに使用するストレージクラスを選択する必要があります。デフォルトでは、VMに名前が生成され、デフォルトのストレージクラスが選択されます。問題がなければ、*Create VirtualMachine* ボタンをクリックします。
+
image::2025_spring/module-07-tempinst/38_VM_Details.png[link=self, window=blank, width=100%]

. 仮想マシンの概要ページに移動し、インスタンスタイプを使用してプロビジョニングされたVMが起動して実行中になっていることを確認します。
+
image::2025_spring/module-07-tempinst/39_VM_Overview.png[link=self, window=blank, width=100%]

[[cleanup]]
== クリーンアップ

次のラボでリソースを節約するには、このモジュールで作成したVMをすべて停止してください。

. 左側のメニューで *Virtualization* パースペクティブに移動し、*Virtualmachines* をクリックします。
. VMワークロードをホストしている、アクセス可能な各プロジェクトが、中央列のツリービューに表示されます。（最低限、プロジェクト *vmimported-{user}* および *vmexamples-{user}* を展開して、仮想マシンのステータスを確認してください。
. VMのステータスが *Running* となっているものがあれば、中央のツリー列でVMをハイライト表示し、 *Actions* ドロップダウンメニューから *Stop* ボタンまたはオプションを選択します。

これで、すべてのVMが *Stopped* 状態になっているはずです。

image::2025_spring/module-07-tempinst/40_All_Stopped.png[link=self, window=blank, width=100%]


== まとめ

このセクションでは、データベースなどの特定のワークロードで使用できるテンプレートを作成するために、既存のテンプレートを複製およびカスタマイズする方法を学びました。また、ブートソースを持たない既存のWindowsテンプレートを構成し、インストールプロセスを自動化する方法も学びました。これにより、そのVMで作成されたインストールPVCをクローン化することで、今後の展開を簡単に作成できるようになります。また、特定のワークロード向けに仮想マシンをさらにカスタマイズし、よりクラウドに近い体験を実現するためのインスタンスタイプの使用方法についてもご紹介しました。