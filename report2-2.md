# 課題2-2：複数スイッチ対応
* 学籍番号：33E16009
* 氏名：坂本 昂輝

## 課題内容
複数スイッチに対応したラーニングスイッチ(multi\_learning\_switch.rb)の動作を説明せよ。

* 複数スイッチのForwarding DBをどのように実装しているか、コードとその動作を説明せよ。
* 動作の様子、フローテーブルの内容もステップ毎に確認せよ。
* 必要に応じて図解せよ。

## 解答
### コード
multi\_learning\_switch.rbにおける複数スイッチのFDBは、@fdbsという連想配列に、各スイッチに対応するFDBインスタンスを入れることで、全てのFDBエントリを一元管理するという方法をとっている。以降では、multi\_learning\_switch.rbに記述された各ハンドラや各メソッドを説明する。

#### fdbの読み込み
```
require 'fdb'
```
fdb.rbでは、MACアドレスとポート番号のペアを保持するデータベースを記述している。この中で、後述するlearn、lookup、ageメソッドが記述されている。


#### タイマーイベントの設定
```
  timer_event :age_fdbs, interval: 5.sec
```
5秒毎に後述のage_fdbs関数を実行する。


#### startハンドラ（プログラム開始時）
```
  def start(_argv)
    @fdbs = {}
    logger.info 'MultiLearningSwitch started.'
  end
```
プログラム開始時にstartハンドラが呼び出される。ここでは、コントローラに配備され、各スイッチのFDBを管理する連想配列@fdbsを作成し、標準出力に"MultiLearningSwitch started."と表示する。


#### switch\_readyハンドラ（スイッチ準備時）
```
  def switch_ready(datapath_id)
    @fdbs[datapath_id] = FDB.new
  end
```
スイッチ準備時にswitch\_readyハンドラが呼び出される。ここでは、startハンドラで作成しておいた@fdbsに、そのスイッチのdpidをキーとして、そのスイッチ用のFDB(インスタンス)を作成して登録する。


#### packet\_inハンドラ（packet\_inが発生した時）
```
  def packet_in(datapath_id, message)
    return if message.destination_mac.reserved?
    @fdbs.fetch(datapath_id).learn(message.source_mac, message.in_port)
    flow_mod_and_packet_out message
  end
```
送信先のMACアドレスが、送信元のホストが繋がるスイッチのフローテーブルになく、packet\_inが発生した時にpacket\_inハンドラが呼び出される。ここでは、MACアドレスが802.1~の予約済みMACアドレスかどうかを判定し、予約済みであればそのパケットを削除する。そうでなければ、送信元のdpidをキーとしてFDBインスタンスを取り出し、そのFDBに送信元のMACアドレスとポート番号を登録する。最後に、後述するflow\_mod\_and\_packet\_outメソッドにmessageを渡して終了する。

#### age\_fdbsメソッド
```
  def age_fdbs
    @fdbs.each_value(&:age)
  end
```
本メソッドは、タイマーイベント設定により5秒毎に呼び出される。ここでは、全てのFDBエントリが最新の状態を保っているかを確認し、古いエントリであれば削除する。具体的には、@fdbsに登録されている全てのFDBエントリに対し、現在の時刻から最後にそのFDBエントリを更新した時刻を引き、その値がage_max(default:300)を越えれば古いと判定される。

#### flow\_mod\_and\_packet\_outメソッド
```
  def flow_mod_and_packet_out(message)
    port_no = @fdbs.fetch(message.dpid).lookup(message.destination_mac)
    flow_mod(message, port_no) if port_no
    packet_out(message, port_no || :flood)
  end
```
本メソッドは、packet\_inハンドラの最後で呼び出される。ここでは、引数で受け取ったパケットのdpidを格納しているFDBにおいて、送信先のMACアドレスが登録されているかを確認し、されていたらport_noにポート番号が返される。もし登録されていれば、後述のflow\_modメソッドにより、該当スイッチのフローテーブルにそのMACアドレスとポート番号が記録される。そうでなければ、何もしない。最後に、後述のpacket\_outメソッドに対して、あるポートにパケットを流すのか、もしくはフラッディングさせるのかに関して引数によって指示を出す。

#### flow\_modメソッド
```
  def flow_mod(message, port_no)
    send_flow_mod_add(
      message.datapath_id,
      match: ExactMatch.new(message),
      actions: SendOutPort.new(port_no)
    )
  end
```
本メソッドでは、引数で受け取ったパケットとポート番号を用いて、send\_flow\_mod\_add命令を出す。send\_flow\_mod\_addで、フローテーブルにホストのMACアドレスと対応するポート番号がセットで登録される。

#### packet\_outメソッド
```
  def packet_out(message, port_no)
    send_packet_out(
      message.datapath_id,
      packet_in: message,
      actions: SendOutPort.new(port_no)
    )
  end
```
本メソッドでは、引数で受け取ったパケットとポート番号を用いて、send\_flow\_mod\_add命令を出す。第二引数にポート番号が指定されていれば、そのポートにパケットを送るが、:floodが指定されていれば、送信元以外のポート全てにパケットを送る。

### 動作確認
仮想スイッチを2つ(vswitch1, vswitch2)用意し、それぞれのスイッチに仮想ホストを1つずつ割り当てた(vswitch1のポート1にvhost1, vswitch2のポート4にvhost2)。そのようなネットワークにおいて、以下の順序でパケットを送信し、動作確認を行った。

1. vhost1 -> vhost2
2. vhost2 -> vhost1
3. vhost1 -> vhost2

各段階でtrema dump\_flowsコマンドにより、vswitch1のフローテーブルを見ることで、段階3において初めてフローエントリが作成されることを確認した。


## 参考文献
* [message.destination_mac.reserved?](https://github.com/yasuhito/trema-book/issues/90)