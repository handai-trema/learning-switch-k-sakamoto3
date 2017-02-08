# 課題4-2：マルチプルテーブルを読む
* 学籍番号：33E16009
* 氏名：坂本 昂輝

## 課題内容
OpenFlow 1.3 版スイッチの動作を説明しよう。

スイッチ動作の各ステップについて、trema dump_flows の出力 (マルチプルテーブルの内容) を混じえながら動作を説明すること。

### 実行のしかた
以下のように `--openflow13` オプションが必要です。

```
$ bundle exec trema run lib/learning-switch13.rb --openflow13 -c trema.conf
```

## 解答

### ネットワーク構成

使用するネットワーク構成は以下の通りである。

```
vswitch('lsw') {
  datapath_id 0xabc
}

vhost ('host1') {
  ip '192.168.0.1'
}

vhost ('host2') {
  ip '192.168.0.2'
}

link 'lsw', 'host1'
link 'lsw', 'host2'
```


### 動作シナリオ

想定する動作シナリオは以下の通りである。

1. 初期フローテーブルの表示
2. host1からhost2へパケット送信かつフローテーブルの表示
3. host2からhost1へパケット送信かつフローテーブルの表示

### 動作説明

動作シナリオの順に沿って、フローテーブルの内容を交えながらOpenFlow1.3版スイッチの動作を説明する。

#### シナリオ１：初期フローテーブルの表示

tremaを起動した時の初期フローテーブルは以下の通りである。

```
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=18.685s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=18.651s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=18.651s, table=0, n_packets=0, n_bytes=0, priority=1 actions=goto_table:1
cookie=0x0, duration=18.651s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=18.651s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535
```

OpenFlow1.3版では初期フローテーブルは2種類ある。テーブル0がフィルタリング処理を担い、マッチしたパケットをドロップする。フィルタリングにかからなければテーブル1に移り、マッチしたパケットに対してなんらかの処理を行う。以下で各行の意味を説明する。

３行目では、テーブル１に移ることが記載されている。そのため、１、２行目では３行目の優先度１よりも高い優先度２を設定し、テーブル０のフィルタリング処理を記述している。１、２行目ではマルチキャストのパケットをドロップする。

４行目では、宛先MACアドレスがブロードキャストアドレスの場合、パケットをフラッディングする。そうでなければ、５行目でパケットインする。５行目でパケットインするため、新たにフローエントリを追加する際には、５行目の優先度１よりも高い値を設定してテーブル１に追加されていく。

#### シナリオ２：host1からhost2へパケット送信かつフローテーブルの表示

実行結果は以下の通りである。

```
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=36.998s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=36.964s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=36.964s, table=0, n_packets=1, n_bytes=42, priority=1 actions=goto_table:1
cookie=0x0, duration=36.964s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=36.964s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535
```

結果として、フローエントリの追加は見られないが、３行目と５行目のn\_packetsとn\_bytesに変化が見られる。その理由は、send\_packetsはブロードキャストではないため、１・２行目にマッチせず、３行目でテーブル１に移っているため、３行目のn\_packetsが1増加し、n\_bytesが42（中身はないため、おそらくヘッダ部分のみでこのデータサイズ）増加している。

また、宛先MACアドレスがブロードキャストアドレスでもないため、４行目にマッチせず、最終的に５行目でパケットインが発生している。そのため、コントローラはFDBに送信元（host1）のアドレスを記憶し、パケットをフラッディングさせていると推測できる。したがって、次にhost2からhost1へパケットを送信すると、優先度２以上のフローエントリが追加されると予測できる。

#### シナリオ３：host2からhost1へパケット送信かつフローテーブルの表示

実行結果は以下の通りである。

```
$ ./bin/trema send_packets --source host2 --dest host1
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=61.281s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=61.247s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=61.247s, table=0, n_packets=2, n_bytes=84, priority=1 actions=goto_table:1
cookie=0x0, duration=61.247s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=21.986s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=6c:f7:e2:84:f3:24,dl_dst=62:a3:8b:6d:48:9d actions=output:1
cookie=0x0, duration=61.247s, table=1, n_packets=2, n_bytes=84, priority=1 actions=CONTROLLER:65535
```

新たなテーブルの５行目に期待通りのフローエントリが追加された。これは、ポート２から入ってきたパケットの送信元MACアドレスが6c:f7:e2:84:f3:24（host1）かつ宛先MACアドレスが62:a3:8b:6d:48:9d（host2）であった場合、ポート１へパケットアウトするという意味である。また、この行のn\_packetsに変化がないことから、今回送信したパケットはコントローラが直接処理したものと考えられる。

以上が簡単なマルチプルテーブルの内容の変化を交えたOpenFlow1.3版のスイッチの動作である。