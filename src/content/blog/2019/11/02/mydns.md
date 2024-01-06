---
title: 'MyDNSのIPアドレス通知スクリプト'
description: 'MyDNS.jpにIPアドレスを通知するスクリプトを定期的に実行する仕組みをArchLinux上に構築する'
pubDate: '2019-11-02T23:26+09:00'
---

MyDNS.jpにIPアドレスを通知するスクリプトを定期的に実行する仕組みをArchLinux上に構築する

1. 通知スクリプト作成

    /root/mydns_update.sh

    ``` bash
    #!/bin/sh
    USER=ユーザ名
    PASSWORD=パスワード

    curl --user $USER:$PASSWORD --silent -o /dev/null http://www.mydns.jp/login.html
    curl --user $USER:$PASSWORD --silent -o /dev/null http://ipv4.mydns.jp/login.html
    ```

1. 作成したスクリプトに実行権限をつける

    ``` bash
    $ sudo chmod +x /root/mydns_update.sh
    ```

1. Arch Linuxの場合cronは非推奨らしいので、systemd/Timersを使ってスクリプトを毎日実行させるようにする

    /etc/systemd/system/mydns.service

    ``` ini
    [Unit]
    Description=MyDNS dnsinfo update
    After=network.target remote-fs.target nss-lookup.target

    [Service]
    Type=oneshot
    ExecStart=/root/mydns_update.sh
    ```

    /etc/systemd/system/mydns.timer

    ``` ini
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
    ```

1. サービスを有効化する

    ``` bash
    $ sudo systemctl daemon-reload
    $ sudo systemctl start mydns.timer
    $ sudo systemctl enable mydns.timer
    ```
