---
title: 'MinimServer導入メモ'
description: 'ArchLinux上でMinimServerコンテナを動かす'
pubDate: '2024-03-17T20:00+09:00'
---
OpenHomeサーバーとして[readyMedia(旧MiniDLNA)](https://sourceforge.net/projects/minidlna/)を使ってネットワークオーディオ環境を構築していたのだけど

- 不具合があっても修正されるのに時間が掛かる
- 時々クライアントアプリが接続エラーを返す時がある
- パッチを当てないと表示されるカバー画像が160x160と小さい
- アルバムアーティスト情報しか送ってくれないのでコンピレーションアルバムとかだと
- 複数枚CDの場合ディスク番号順→トラック番号順にソートしてくれない

等、不満が色々あったので[MinimServer](https://minimserver.com/)に乗り換えてみたので手順をメモ

1. MinimServerをコンテナ上で動かす

    自宅サーバーに入れているのがPodmanなのでPodmanを使っているが、Dockerでもほぼ同じ手順でできるはず

    1. イメージを落としてくる

        ```
        sudo podman pull docker.io/minimworld/minimserver:2.2
        ```

    1. コンテナを起動する

        --envオプションで「TZ=Asia/Tokyo」を付けないとサーバーのタイムゾーンがGMTになるらしいので付けておく

        ```
        sudo podman run -d --name minimserver --network host \
        --env TZ=Asia/Tokyo \
        --mount type=bind,src=曲ファイルが置いてあるディレクトリ,dst=/Music,readonly \
        minimworld/minimserver:2.2
        ```


1. 自動起動設定

    サーバーを再起動した時、MinimServerも自動的に起動して欲しいのでsystemdにサービスとして登録する

    1. 起動/停止用スクリプトを作成する

        /usr/local/bin/minimserver

        ``` bash
        #!/usr/bin/env bash
        if [ "$(id -u)" -ne 0 ]; then
            echo -e "non-root user"
            exit 1
        fi

        if [ "$1" = "stop" ]; then
            /usr/bin/podman stop -t 10 minimserver
            exit 0
        fi

        /usr/bin/podman ps -a --format "{{.Names}}" | grep -x minimserver > /dev/null
        if [ $? -eq 0 ]; then
            # コンテナが存在する場合はそれを使う
            /usr/bin/podman start -a minimserver
            exit 0
        fi

        # コンテナが無かったら新しく作る
        /usr/bin/podman run -d --name minimserver --network host \
        --env TZ=Asia/Tokyo \
        --mount type=bind,曲ファイルが置いてあるディレクトリ,dst=/Music,readonly \
        minimworld/minimserver:2.2
        ```

    1. 起動/停止用スクリプトに実行権限をつける

        ``` bash
        sudo chmod +x /usr/local/bin/minimserver
        ```

    1. サービス設定を作成する

        /etc/systemd/system/minimserver.service

        ``` ini
        [Unit]
        Description=MinimServer
        Wants=syslog.service
        After=network.target remote-fs.target nss-lookup.target
        [Service]
        Restart=always
        ExecStart=/usr/local/bin/minimserver start
        ExecStop=/usr/local/bin/minimserver stop
        [Install]
        WantedBy=multi-user.target
        ```


    1. サービスを有効化する

        ``` bash
        sudo systemctl daemon-reload
        sudo systemctl enable minimserver
        ```


1. ファイヤーウォール設定変更

    Firewalldが有効になっているとDLNAクライアントと通信できないので設定を変更する  
    解放する必要があるのはTCP9790-9791ポートとUDP1900ポート

    1. サービス設定を作成する

        /etc/firewalld/services/minimserver.xml

        ``` xml
        <?xml version="1.0" encoding="utf-8"?>
        <service>
        <short>MinimServer</short>
        <description>MinimServer is a UPnP music server with a number of innovative features that make it easier to organize and explore your music collection.</description>
        <include service="ssdp"/>
        <port protocol="tcp" port="9790"/>
        <port protocol="tcp" port="9791"/>
        </service>
        ```

    1. 作成したサービス設定を読み込ませる

        ``` xml
        sudo firewall-cmd --reload
        ```

    1. サービス設定を有効にする

        自分の場合LAN内通信のファイヤーウォール設定をhomeゾーンで管理しているため「--zone=home」を指定している  

        ``` xml
        sudo firewall-cmd --permanent --zone=home --add-service=minimserver
        sudo firewall-cmd --reload
        ```
