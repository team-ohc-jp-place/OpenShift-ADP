
# 2. iPXE 環境の作成
`iPXE`というネットワークブートの方法を使用して、`Bootstrap` / `Master` / `Worker` ノードの RHEL `CoreOS` をインストールしていきます。

(`ISOイメージ`による `CoreOS` インストールを選択した場合は、この手順は必要ありません)

当初は`ISOイメージ`によるインストールを行おうとしていたのですが、リモートワークの環境では何故かブート時のコンソールが上手く操作できない。と言う環境上の問題に出くわして、`iPXE`での環境構築を行う事になりました。

当初は想定していなかったのですが、この後、環境を破壊して、全ての`ノード`を作り直さなければいけなくなる事が何度もありました。その時に、`iPXE`によるネットワークインストールにより、かなり時間を節約する事ができました。

実運用を考えると`iPXE`等によるネットワークブートでOSをインストールする環境を作成しておく事をおすすめします。

##2.1.必要なサーバーのインストール

以下を参考にして環境を構築します。

<a href="https://qiita.com/Yuhkih/items/c7cc9978ee107784c97f">iPXE ブート環境をセットアップする</a>

リンク先の汎用的な `iPXE`環境のサーバー構成は、この手順で想定しているサーバー構成と同じです。

##2.2.DHCPサーバーを使った固定IPアドレスの割り振り

汎用的な手順で`iPXE`環境を構築した後、`OpenShift`クラスターをインストールするためのカスタマイズを施します。

`UPI`インストールでは、事前にロードバランサーに `BootStratp` / `Master` / `Workerノード`を登録しておく必要があるため、それぞれの固定のIPが必要になります。

`/etc/dhcp/dhcpd.conf` を編集して、ノードの `MACアドレス`を基準して固定IPを割り振る設定をします。

どの部分を追加したかは設定ファイルの中にコメントで区切っています。

```/etc/dhcp/dhcpd.conf

<省略> 

 # PXE Boot  -  start
  class "pxeclients" {
      match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
      next-server 192.2.0.101;  # TFTP Server address

      if option arch = 00:07 { # x86-64

            # Chain loading.
            # 1st time, tell iPXE image place as boot file on TFTP server.
            # 2nd time, tell iPXE script place on HTTP server.

            if exists user-class and option user-class = "iPXE" { #  the 2nd time. Go to HTTP Server to get iPXE script
                filename "http://192.2.0.101/pxeboot.ipxe";
            } else {  # the 1st time. Go to TFTP to get iPXE.
                filename "efi/ipxe.efi";
                # filename "undionly.kpxe";
            }

      } else {
        # filename "pxelinux/pxelinux.0";
        filename "pxelinux.cfg/default";
        # filename "menu/boot.ipxe";
      }


 }
 # PXE Boot - end

 #  ここから MACアドレスに対して固定 IPを割り振る設定
 host bs{
                hardware ethernet 00:50:56:96:11:c2;
                fixed-address 192.2.0.11;
 }
 host m1{
                hardware ethernet 00:50:56:96:37:c8;
                fixed-address 192.2.0.21;
 }
 host m2{
                hardware ethernet 00:50:56:96:ec:a7;
                fixed-address 192.2.0.22;
 }
 host m3{
                hardware ethernet 00:50:56:96:e5:2f;
                fixed-address 192.2.0.23;
 }

 host w1{
                hardware ethernet 00:50:56:96:6b:da;
                fixed-address 192.2.0.31;
 }

 host w2{
                hardware ethernet 00:50:56:96:c1:c5;
                fixed-address 192.2.0.32;
 }
 host w3{
                hardware ethernet 00:50:56:96:e3:f5;
                fixed-address 172.16.0.33;
 }
}
``` 

`MACアドレス`は、実際に`CoreOS`のインストール対象のサーバーの`NIC`の`MACアドレス`を確認し、実際の環境にあわせて書き替えて下さい。


## 2.3.CoreOSのインストールイメージを入手する

`iPXE`ブートで使用する`CoreOS`のファイルを手に入れます。

以下より 
https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/

1)compressed metal RAW image
2)knernel 
3)initramfs 

をダウンロードします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/99425/09afdf44-cf8b-8024-7de7-b53cf3dc1769.png)

似たような名前のファイルが幾つかありますが、以下のフォーマットで名前が付いているものをダウンロードします。

```
Compressed metal RAW image: rhcos-<version>metal.<architecture>.raw.gz
kernel: rhcos-<version>-installer-kernel-<architecture>
initramfs: rhcos-<version>-installer-initramfs.<architecture>.img
```

ここでは、以下のファイルをダウンロードします。

```
rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img
rhcos-4.5.6-x86_64-installer-kernel-x86_64
rhcos-4.5.6-x86_64-metal.x86_64.raw.gz
```

※`ISO`ファイルである`rhcos-<version>-installer.<architecture>.iso`は、`ISO`ブートで `CoreOS`をインストールする時に使用します。この手順(`iPXE`ブート)では使用しません。

#### 2.2.1.ダウンロードしたファイルを HTTP サーバー(踏み台サーバー)にコピーする

これらのファイルも `HTTP サーバー`(`踏み台サーバー`に同じ) のディレクトリに配置します。ここでは nginx のルート(`/usr/share/nginx/html`)ディレクトリに配置しています。

```
cp * /usr/share/nginx/html
```

#### 2.2.2.ファイルのアクセスを確認する。

`HTTP サーバー`上のファイルがアクセスできるか確認します。
以下のコマンドで HTTP 200 が返ってくれば OK です。

```
[root@bastion ~]#  curl -D - -s -o /dev/null http://bastion.example.localdomain/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img
[root@bastion ~]#  curl -D - -s -o /dev/null http://bastion.example.localdomain/rhcos-4.5.6-x86_64-installer-kernel-x86_64
[root@bastion ~]# curl -D - -s -o /dev/null http://bastion.example.localdomain/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz
```

`bastion.example.localdomain`は、この環境での`HTTP サーバー`(=`踏み台サーバー`)のホスト名です。

## 2.4.iPXEスクリプトの作成

`iPXE`インストール時の作業を記した`iPXE`スクリプトを作成します。内容は<a href="https://qiita.com/Yuhkih/items/c7cc9978ee107784c97f">iPXE ブート環境をセットアップする</a>のサンプルと同じものです。

```
/usr/share/nginx/html/pxeboot.ipxe
#!ipxe

# dhcp
# Some menu defaults
set menu-timeout 300000
isset ${menu-default} || set menu-default exit


:start

menu Please choose an type of node you want to install
item --gap --           -------------------------- node type -------------------------
item --key b bootstrap  Install Bootstrap Node
item --key m master     Install Master Node
item --key w worker     Install Worker Node
item --gap --           -------------------------- Advanced Option --------------------
item --key c config     Configure settings
item shell              Drop to iPXE shell
item reboot             Reboot Computer
choose --timeout ${menu-timeout} --default ${menu-default} selected || goto cancel
goto ${selected}

:bootstrap   # ここはインストールしたい OS毎に異なる
kernel  http://192.2.0.101/rhcos-4.5.6-x86_64-installer-kernel-x86_64 ip=dhcp rd.neednet=1 initrd=rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.install_dev=sda coreos.inst.image_url=http://192.2.0.101/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.2.0.101/bootstrap.ign
initrd http://192.2.0.101/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img
boot

:master　  # ここはインストールしたい OS毎に異なる
kernel  http://192.2.0.101/rhcos-4.5.6-x86_64-installer-kernel-x86_64 ip=dhcp rd.neednet=1 initrd=rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.install_dev=sda coreos.inst.image_url=http://192.2.0.101/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.2.0.101/master.ign
initrd http://192.2.0.101/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img
boot

:worker　  # ここはインストールしたい OS毎に異なる
kernel  http://192.2.0.101/rhcos-4.5.6-x86_64-installer-kernel-x86_64 ip=dhcp rd.neednet=1 initrd=rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.install_dev=sda coreos.inst.image_url=http://192.2.0.101/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.2.0.101/worker.ign
initrd http://192.2.0.101/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img
boot

:exit
exit

:cancel
echo You cancelled the menu, dropping you to a shell

:shell
echo Type 'exit' to get the back to the menu
shell
set menu-timeout 0
goto start

:reboot
reboot

:exit
exit
```

最終的な上記のようなスクリプトを作成すれば良いのですが、各ラベル毎の解説をします。

### 2.4.1.BootStrap用の記述

```
:bootstrap  
kernel  http://192.2.0.101/rhcos-4.5.6-x86_64-installer-kernel-x86_64 ip=dhcp rd.neednet=1 initrd=rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.install_dev=sda coreos.inst.image_url=http://192.2.0.101/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.2.0.101/bootstrap.ign
initrd http://192.2.0.101/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img
boot
```

この例で`192.2.0.101`は、`踏み台サーバー`(`HTTPサーバー`/`TFTPサーバー`が同居)の`IPアドレス`です。

`rhcos-4.5.6-x86_64-installer-kernel-x86_64`
`rhcos-4.5.6-x86_64-metal.x86_64.raw.gz`
`rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img`

は、`/usr/share/nginx/html`(Nginxのルート) に配置された`CoreOS`関連のファイル名ですが、実際にダウンロードしてきたファイル名に合わせます。

`bootstrap.ign`は、後の手順で作成し、`/usr/share/nginx/html`に配置します。

※2020/10/06時点でOCP4.5のマニュアルに記載されている iPXE用のスクリプトのサンプルの記述に誤記があり、そのままでは動作しないのでご注意下さい。[Bug 1741922 - [DOCS] Failed to open file error during iPXE install](https://bugzilla.redhat.com/show_bug.cgi?id=1741922)

### 2.4.2.Master用の記述

```
:master
kernel  http://192.2.0.101/rhcos-4.5.6-x86_64-installer-kernel-x86_64 ip=dhcp rd.neednet=1 initrd=rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.install_dev=sda coreos.inst.image_url=http://192.2.0.101/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.2.0.101/master.ign
initrd http://192.2.0.101/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img
boot
```

基本的に `:boostratp`のラベル部分の記述と同じですが、Master 用の ignition ファイル`master.ign`を参照している部分が違います。

`master.ign`は、後の手順で作成し、`/usr/share/nginx/html`に配置します。

### 2.4.3.Worker用の記述

```
:worker　  
kernel  http://192.2.0.101/rhcos-4.5.6-x86_64-installer-kernel-x86_64 ip=dhcp rd.neednet=1 initrd=rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.install_dev=sda coreos.inst.image_url=http://192.2.0.101/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.2.0.101/worker.ign
initrd http://192.2.0.101/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img
boot
```

基本的に `:boostratp`、`:master`のラベル部分の記述と同じですが、Worker 用の ignition ファイル`worker.ign`を参照している部分が違います。

`worker.ign`は、後の手順で作成し、`/usr/share/nginx/html`に配置します。
