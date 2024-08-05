---
title: 'Podman Quadlet導入メモ'
description: 'PodmanコンテナをQuadletで自動起動させる'
pubDate: '2024-08-06T07:00+09:00'
---

[MinimServer導入時](/2024/03/17/minimserver/)、自動起動まわりで色々やるのが面倒だと思っていたけど、
PodmanのQuadletって機能を使えばすっきりするみたいなので試してみた

1. 自動起動設定まわりをリセット

    ``` bash
    sudo systemctl stop minimserver
    sudo systemctl disable minimserver
    sudo rm /etc/systemd/system/minimserver.service
    sudo rm /usr/local/bin/minimserver
    sudo systemctl daemon-reload
    ```

1. Quadletファイル作成

    Containerセクション以外はsystemdのユニットファイルとほぼ同様  
    Containerセクションについては[podman実行時に指定していたオプションを書けばいいみたい](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html#container-units-container)

    /etc/containers/systemd/minimserver.container

    ``` ini
    [Unit]
    Description=MinimServer
    Wants=syslog.service
    After=network.target remote-fs.target nss-lookup.target

    [Container]
    Image=docker.io/minimworld/minimserver:2.2
    Network=host
    Environment=TZ=Asia/Tokyo
    Mount=type=bind,src=曲ファイルが置いてあるディレクトリ,dst=/Music,readonly

    [Service]
    Restart=always

    [Install]
    WantedBy=multi-user.target
    ```

1. 作ったQuadletファイルに問題がないかdryrunでチェック

    ``` bash
    sudo /usr/lib/systemd/system-generators/podman-system-generator --dryrun
    ```

1. サービスを有効化する

    ``` bash
    sudo systemctl daemon-reload
    sudo systemctl enable minimserver
    sudo systemctl start minimserver
    ```

