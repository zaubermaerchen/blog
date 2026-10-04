---
title: '自宅サーバー更改'
description: '自宅サーバーのArch Linuxを再インストールした際の手順'
pubDate: '2026-10-04T22:14+09:00'
---

自宅サーバーをZen2環境からRaptor Lake-S Refresh環境に移行したので再インストールの手順メモ

## スペック

- OS: Arch Linux
- CPU: Intel Core i3 14100
- マザーボード: MSI PRO B760M-A DDR4 II
- メモリ: crucial PC4-25600 CL22 8GBx2(現サーバーから流用)
- システムストレージ: KIOXIA EXCERIA 1TB(メインPCで使っていたものを流用)
- データストレージ: WD Red 3TB(現サーバーから流用)
- 外付けNIC: 玄人志向 GBE2.5-PCIE(現サーバーから流用)

NICは外付けがネットワーク用、オンボードが外部からのSSH接続用として使い分け


## 再インストール

[Arch Wikiのインストールガイド](https://wiki.archlinux.jp/index.php/%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%82%AC%E3%82%A4%E3%83%89)を参考に設定を進める

### 基本設定

1. 日本語キーボードレイアウトに設定

    ``` bash
    loadkeys jp106
    ```

2. システムクロック更新

    ``` bash
    timedatectl status
    ```

### パーティション設定

間違ってデータストレージをフォーマットしないように最初は外した状態で進めていく

1. システムストレージのパーティション

    下記内容でパーティションを分ける

    |領域名|デバイス名|サイズ|
    |--|--|--|
    |EFI領域|/dev/nvme0n1p1|1024MiB|
    |スワップ領域|/dev/nvme0n1p2|4096MiB|
    |ルートディレクトリ|/dev/nvme0n1p3|残りすべて|

    ``` bash
    # パーティション分け
	parted /dev/nvme0n1 --script \
	  mklabel gpt \
	  mkpart ESP fat32 1MiB 1025MiB \
	  set 1 esp on \
	  mkpart swap linux-swap 1025MiB 5121MiB \
	  mkpart root btrfs 5121MiB 100%

	# 確認(1:ESP/2:swap/3:rootがあればOK)
	parted /dev/nvme0n1 print
    ```

1. システムストレージのフォーマット

    ``` bash
	# EFI領域のフォーマット
	mkfs.fat -F 32 -n EFI /dev/nvme0n1p1

	# スワップ領域のフォーマット
    mkswap -L swap /dev/nvme0n1p2

	# root領域はBtrfsでフォーマット
	mkfs.btrfs -L root /dev/nvme0n1p3
    ```

1. システムストレージのマウント

    ``` bash
	# root領域の一時マウント
	mkdir -p /mnt/btrfs
	mount /dev/nvme0n1p3 /mnt/btrfs

	# サブボリューム作成
	btrfs subvolume create /mnt/btrfs/@
	btrfs subvolume create /mnt/btrfs/@snapshots

	# 確認
	btrfs subvolume list /mnt/btrfs

	# 一時マウントを解除
	umount /mnt/btrfs

	# @ を /mnt にマウント
	mount -o subvol=@,noatime,compress=zstd /dev/nvme0n1p3 /mnt

	# @snapshots をマウント
	mkdir -p /mnt/.snapshots
	mount -o subvol=@snapshots,noatime,compress=zstd /dev/nvme0n1p3 /mnt/.snapshots

	# EFI領域のマウント
	mkdir -p /mnt/boot
	mount /dev/nvme0n1p1 /mnt/boot

	# スワップを有効化
	swapon /dev/nvme0n1p2
    ```

1. データストレージを接続するため一旦シャットダウン

    ``` bash
	sudo umount -R /mnt
	sudo poweroff
    ```

1. システムストレージ、データストレージのマウント

    データストレージを接続後、電源投入  
    システムストレージのマウント状態はリセットされているのでこちらも再度設定する

    ``` bash
	# @ を /mnt にマウント
	mount -o subvol=@,noatime,compress=zstd /dev/nvme0n1p3 /mnt

	# @snapshots をマウント
	mkdir -p /mnt/.snapshots
	mount -o subvol=@snapshots,noatime,compress=zstd /dev/nvme0n1p3 /mnt/.snapshots

	# EFI領域のマウント
	mkdir -p /mnt/boot
	mount /dev/nvme0n1p1 /mnt/boot

	# データストレージのHDDパーティションを /mnt/home にマウント
	mkdir -p /mnt/home
	mount /dev/sda1 /mnt/home

	# スワップを有効化
	swapon /dev/nvme0n1p2
    ```

### インストール

1. 必須パッケージと必要そうなパッケージのインストール

    ``` bash
	pacstrap -K /mnt base linux linux-firmware intel-ucode btrfs-progs networkmanager grub efibootmgr openssh git neovim
    ```

### システム設定
1. fstabの生成

    ``` bash
    genfstab -U /mnt >> /mnt/etc/fstab
    ```

1. chroot

    ``` bash
    arch-chroot /mnt
    ```

1. タイムゾーン設定

    ``` bash
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
    hwclock --systohc
    ```

1. ロケール設定

    ``` bash
    sed -i '/^# *en_US.UTF-8 UTF-8/s/^# *//g' /etc/locale.gen
    sed -i '/^# *ja_JP.UTF-8 UTF-8/s/^# *//g' /etc/locale.gen
    locale-gen
    printf 'LANG=en_US.UTF-8\n' > /etc/locale.conf
    printf 'KEYMAP=jp106\n' > /etc/vconsole.conf
    ```

1. ネットワーク設定

    1. ホスト名設定

        ``` bash
        hostnamectl set-hostname ホスト名
        ```

    1. サービスの有効化

        ``` bash
        systemctl enable NetworkManager
        ```

1. Initramfs

    ``` bash
    mkinitcpio -P
    ```

1. Rootパスワード設定

    ``` bash
    passwd
    ```

1. ブートローダー

    GRUBを使う

    ``` bash
	# GRUB のインストール
	grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --removable
	
	# 設定ファイル（grub.cfg）の自動生成
	grub-mkconfig -o /boot/grub/grub.cfg
    ```
		
1. インストール完了

	1. chroot 環境を抜ける

        ``` bash
        exit
        ```
		
	1. マウントしていたパーティションをすべて解除する

        ``` bash
        umount -R /mnt
        ```

### 各種サービス有効化

1. ネットワーク有効化
    1. `enp4s0`設定

        `enp4s0`(外付け)を通常通信用として使用するように設定
    
        ```bash
        nmcli connection add \
            type ethernet \
            ifname enp4s0 \
            con-name main \
            ipv4.method auto \
            ipv4.route-metric 100 \
            ipv6.method auto \
            ipv6.route-metric 100
        ```	

    1. `enp3s0`設定

        `enp3s0`(オンボード)はリモートからのSSH接続用として使用するように設定  
        なお、前提条件は下記の通り

        - ルーター側で`enp3s0`のMACアドレスに固定IPアドレスが割り当てられている状態
        - 割り当てられているIPアドレスのサブネットは外部接続用(`192.168.101.0/24`)

        1. DHCPとルーティングテーブルの設定

            ``` bash
            nmcli connection add \
                type ethernet \
                ifname enp3s0 \
                con-name management \
                ipv4.method auto \
                ipv4.route-table 101 \
                ipv6.method link-local
            ```	

        1. DHCPから自動取得するDNSサーバ情報を無効化

            `enp3s0`のDHCPからDNS情報を受け取る必要は基本的にないと思うので無視するようにする

            ``` bash
            nmcli connection modify management \
                ipv4.ignore-auto-dns yes
            ```	

        1. Source policy routing

            外部接続用サブネットを送信元とする通信はtable 101を使わせる

            ```
            nmcli connection modify management \
                ipv4.routing-rules \
                "priority 100 from 192.168.101.0/24 table 101"
            ```

        1. 外部接続用サブネットをmain tableにも追加

            `ipv4.route-table=101`にするとDHCP由来のconnected routeもtable 101へ入るため、main tableにも明示的に追加

            ```
            nmcli connection modify management \
                +ipv4.routes \
                "192.168.101.0/24 0.0.0.0 1002 table=254"
            ```

    1. NICとゾーンの紐づけ

        ``` bash
        nmcli connection modify main connection.zone home
        nmcli connection modify management connection.zone public
        ```

    1. 接続を有効化

        ``` bash
        nmcli connection up main
        nmcli connection up management
        ```

1. パッケージインストール

    共通で使うパッケージを入れておく

    ```bash
	pacman -S base-devel ethtool curl logrotate bash-completion less man-db man-pages usbutils
	```

2. firewalld

    1. インストールと有効化

        ```bash
        pacman -S firewalld
        systemctl enable --now firewalld
        ```
        
    1. 設定されているか確認

        NetworkManager側でNICとゾーンの紐づけはされているはずなので確認
    
        ``` bash
        firewall-cmd --get-active-zones
        firewall-cmd --list-all --zone=public
        firewall-cmd --list-all --zone=home
        ```

3. ユーザー作成

    既存`/home`に合わせてユーザーを再作成する

	1. ユーザーが使うパッケージのインストール

        ```
        pacman -S zsh ghostty-terminfo sheldon starship zoxide bat eza ripgrep skim zellij github-cli mise
        ```

	1. 前の環境で使用していたGID/UIDが空いていることを確認

        今回作るユーザーはGID/UIDが共に1000なのでこんな感じ

        ```bash
        getent passwd 1000
        getent group 1000
        ```

	    新規環境なので通常は何も表示されないはず

    1. ユーザー作成

        確認したGID/UIDを使ってユーザーを作成

        ```bash
        groupadd -g 1000 ユーザー名
        useradd -u 1000 -g 1000 -d /home/ユーザー名 -s /usr/bin/zsh -M ユーザー名
        passwd ユーザー名
        ```

    1. sudoを使えるようにする

        ```bash
        usermod -aG wheel ユーザー名
        EDITOR=vi visudo
        ```

	    `visudo`で`%wheel ALL=(ALL:ALL) ALL`を有効化する
	

1. OpenSSH

	1. デフォルト設定をオーバライドする

        ```bash
        mkdir -p /etc/ssh/sshd_config.d
        cat > /etc/ssh/sshd_config.d/10-local.conf <<'EOF'
        PermitRootLogin no
        PasswordAuthentication no
        EOF
        ```


	1. 設定を検証して問題なければ有効化する

        ```bash
        sshd -t
        systemctl enable --now sshd.service
        ```
	
1. Samba
    1. インストール

        SambaをWindowsから検出させたいのでwsddも

        ```bash
        pacman -S samba wsdd
        ```
        

    1. デフォルトの設定ファイルをダウンロード

        Arch Linuxではパッケージを入れてもSambaの設定ファイルが生成されないのでダウンロード

        ```bash
        curl "https://git.samba.org/samba.git/?p=samba.git;a=blob_plain;f=examples/smb.conf.default;hb=HEAD" -o /etc/samba/smb.conf
        ```

    1. 設定ファイルの編集

        `/home/public`をパスワードなしでアクセスできるようにしたいので`/etc/samba/smb.conf`を編集する

        ``` diff
        47a48
        >    server min protocol = SMB2
        51a53
        >    map to guest = Bad User
        55c57
        <    log file = /usr/local/samba/var/log.%m
        ---
        >    log file = /var/log/samba/%m.log
        202,207c204,209
        < ;[public]
        < ;   path = /usr/somewhere/else/public
        < ;   public = yes
        < ;   only guest = yes
        < ;   writable = yes
        < ;   printable = no
        ---
        > [public]
        >    path = /home/public
        >    public = yes
        >    guest only = yes
        >    writable = yes
        >    printable = no
        ```

    1. サービスの有効化と起動

        ```bash
        sudo systemctl enable --now smb.service
        sudo systemctl enable --now wsdd.service
        ```

    1. firewalldの設定更新
	
        homeゾーンでSambaとwsddの通信を許可

        ``` bash
        firewall-cmd --permanent --zone=home --add-service=wsdd --add-service=samba
        firewall-cmd --reload
        firewall-cmd --list-all --zone=home
        ```

    1. Sambaパスワード設定

        WindowsやMacからユーザーのホームディレクトリへアクセスできるようにしたいのでSambaパスワードを設定

        ```
        smbpasswd -a ユーザー名
        ```

1. Avahi

    1. インストール

        ```bash
	    pacman -S avahi
        ```

    1. `enp4s0`のみ有効にする

        ```bash
        sed -i 's/#allow-interfaces=eth0/allow-interfaces=enp4s0/' /etc/avahi/avahi-daemon.conf
        ```
	
    1. Samba用設定ファイルを追加

        SambaをmDNSでMacから検出できるようにしたいので下記設定を追加

        ``` bash
        cat > /etc/avahi/services/smb.service <<'EOF'
        <?xml version="1.0" standalone='no'?>
        <!DOCTYPE service-group SYSTEM "avahi-service.dtd">
        <service-group>
            <name replace-wildcards="yes">%h</name>
            <service>
                <type>_smb._tcp</type>
                <port>445</port>
            </service>
            <service>
                <type>_device-info._tcp</type>
                <port>0</port>
                <txt-record>model=Xserve</txt-record>
            </service>
        </service-group>
        EOF
        ```

    1. サービスの有効化と起動

        ```bash
        systemctl enable --now avahi-daemon.service
        ```
	
1. Podman

    1. インストール

        ``` bash
        pacman -S podman
        ```
	
	    oci-runtimeとしてどれを使うか聞かれたらデフォルト値の`crun`を設定
	
1. Minimserver

	1. Podman Quadlet構成ファイル作成
        ``` bash
        cat > /etc/containers/systemd/minimserver.container <<'EOF'
        [Unit]
        Description=MinimServer
        Wants=syslog.service
        After=network.target remote-fs.target nss-lookup.target

        [Container]
        Image=docker.io/minimworld/minimserver:2.2
        Network=host
        Environment=TZ=Asia/Tokyo
        Mount=type=bind,src=/home/public/Music,dst=/Music,readonly
        Mount=type=bind,src=/var/lib/minimserver,dst=/opt/minimserver/data

        [Service]
        Restart=always

        [Install]
        WantedBy=multi-user.target
        EOF
        ```

	1. 一回起動させる

        ``` bash
        mkdir -p /var/lib/minimserver
        systemctl daemon-reload
        systemctl start minimserver.service
        ```
	
	1. `enp4s0`のサブネットを使いたいので設定して再起動

        ``` bash
        subnet=$(ip -4 route show dev enp4s0 scope link | awk '{sub(/\/.*/, "", $1); print $1; exit}')
        podman exec systemd-minimserver /opt/minimserver/bin/mscript -c "prop ohnet.subnet=$subnet"
        systemctl restart minimserver.service
        ```
	
	1. firewalldのminimserver用のサービスを追加

        ``` bash
        cat > /etc/firewalld/services/minimserver.xml <<'EOF'
        <?xml version="1.0" encoding="utf-8"?>
        <service>
        <short>MinimServer</short>
        <description>MinimServer is a UPnP music server with a number of innovative features that make it easier to organize and explore your music collection.</description>
        <include service="ssdp"/>
        <port protocol="tcp" port="9790"/>
        <port protocol="tcp" port="9791"/>
        </service>
        EOF
        firewall-cmd --reload
        ```
	
    1. firewalldの設定更新

        homeゾーンでminimserverの通信を許可

        ``` bash
        firewall-cmd --permanent --zone=home --add-service=minimserver
        firewall-cmd --reload
        firewall-cmd --list-all --zone=home
        ```
	
1. UPS/Time/Sensorsの有効化

	1. インストール

        ``` bash
	    pacman -S apcupsd chrony lm_sensors
        ```
	
    1. サービスの有効化と起動

        ``` bash
        systemctl enable --now apcupsd.service
        systemctl enable --now chronyd.service
        systemctl enable --now lm_sensors.service
        ```
	
	1. `/etc/conf.d/lm_sensors` の生成

        ``` bash
        sensors-detect
        ```

1. MyDNS

	定期的にMyDNSのIPアドレス通知を実行したいので
	
    1. 認証情報作成

        MyDNSのユーザー名/ログインパスワードを記述する

        ``` bash
        mkdir -p /root/.config/mydns
        chmod 700 /root/.config/mydns
        cat > /root/.config/mydns/credentials <<'EOF'
        MYDNS_USER='xxxxx'
        MYDNS_PASSWORD='xxxxx'
        EOF
        chmod 600 /root/.config/mydns/credentials
        ```

    1. IPアドレス通知スクリプト作成

        ``` bash
        mkdir -p /root/.local/bin
        cat > /root/.local/bin/mydns_update.sh <<'EOF'
        #!/bin/sh
        . /root/.config/mydns/credentials

        #curl --interface enp3s0 --user "$MYDNS_USER:$MYDNS_PASSWORD" --silent -o /dev/null http://www.mydns.jp/login.html
        curl --interface enp3s0 --user "$MYDNS_USER:$MYDNS_PASSWORD" --silent -o /dev/null http://ipv4.mydns.jp/login.html
        EOF
        chmod +x /root/.local/bin/mydns_update.sh
	    ```

    1. systemd用設定ファイルを追加

        IPアドレスの通知を自動的に毎日行うようにしたいので設定

        ``` bash
        cat > /etc/systemd/system/mydns.service <<'EOF'
        [Unit]
        Description=MyDNS dnsinfo update
        After=network.target remote-fs.target nss-lookup.target

        [Service]
        Type=oneshot
        ExecStart=/root/.local/bin/mydns_update.sh
        EOF

        cat > /etc/systemd/system/mydns.timer <<'EOF'
        [Unit]
        Description=MyDNS dnsinfo update
        Wants = multi-user.target
        After = multi-user.target

        [Timer]
        OnCalendar=daily
        RandomizedDelaySec=5s
        Persistent=true
        
        [Install]
        WantedBy=timers.target
        EOF
        chmod +x /root/.local/bin/mydns_update.sh
        ```

    1. サービスの有効化と起動

        ``` bash
        systemctl daemon-reload
        systemctl enable --now mydns.timer
        ```
