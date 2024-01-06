---
title: 'rargs'
description: 'rargsというxargsとawkを合わせたようなツールについて'
pubDate: '2019-01-20T11:06+09:00'
---

[rargs](https://github.com/lotabout/rargs)というxargsとawkを合わせてパターンマッチングを使えるようにしたRust製コマンドラインツールを見つけたけど便利そう。

GitHubのREADME.mdにはcargo使ったインストール方法しか書いていないけど、[Homebrewのパッケージもある](https://formulae.brew.sh/formula/rargs)のでMacだったらbrew install rargsでインストール可能。

[モバマスのカードリストを取得する自作ツール](https://github.com/zaubermaerchen/imas_cg_idol_detail_list)を使って保存したJSONファイルからアイドルリストに登録するのに必要なデータを抽出したい場合、今まではこんなワンライナー書いていてawkで指定するシングル/ダブルクォーテーションの組み合わせで混乱してたけど

``` bash
$ cat アイドル名一覧テキストファイル | peco --exec 'awk -F : '"'"'{print "data/" $0 ".json"}'"'"' | xargs cat | jq -r -c ".[] | [.profile.card_id, .profile.card_name] | @tsv" | less'
```

rargs使うとこんな感じに直感的に書けて大分スッキリした。

``` bash
$ cat アイドル名一覧テキストファイル | peco --exec 'rargs cat data/{0}.json | jq -r -c ".[] | [.profile.card_id, .profile.card_name] | @tsv" | less'
```

Windows(PowerShell)上だと例外吐いて使えないみたいのが残念。
