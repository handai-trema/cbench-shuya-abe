#レポート課題2-1

氏名: 阿部修也  

##課題内容
Rubyのプロファイラを用いてCbenchのボトルネックを解析せよ。
また、余裕のあるものはボトルネックを改善してCbenchを高速化せよ（発展課題）

##課題解答
###ボトルネック解析
Ruby標準のプロファイラであるprofileライブラリを用いてプロファイルを行い、Cbenchのボトルネックを解析した。

####profileライブラリの利用
profileライブラリはRubyの標準的なライブラリとして組み込まれているため、その他のライブラリの利用方法と同様に、以下をソースコードの最初に書くことで読み込める。

```
require 'profile'
```

profileライブラリは、読み込まれたプログラム全体に作用し、各メソッドの実行時間や呼び出し回数などの統計を出力する。

ただし、profileライブラリはそれ自体がオーバーヘッドになることから、このライブラリを利用することでCbenchの結果は悪化する（今回はボトルネックを調べることが目的なので問題はない）。

####プロファイル結果と考察
以下にプロファイルの結果（上位15メソッド）を示す。
結果全体は[profile_org.txt](https://github.com/handai-trema/cbench-shuya-abe/blob/master/profile_org.txt)に保存されている。


```
  %   cumulative   self              self     total
 time   seconds   seconds    calls  ms/call  ms/call  name
199.95   130.67    130.67        2 65335.00 65335.00  IO.select
 99.98   196.01     65.34        2 32670.00 32670.00  Thread#join
 99.98   261.35     65.34        2 32670.00 32670.00  TCPServer#accept
 99.94   326.66     65.31      115   567.91   567.91  Kernel#sleep
  3.24   328.78      2.12    31542     0.07     0.10  Kernel#dup
  2.88   330.66      1.88    40593     0.05     0.31  BinData::Struct#instantiate_obj_at
  2.66   332.40      1.74    47187     0.04     0.11  BinData::BasePrimitive#method_missing
  2.57   334.08      1.68    59181     0.03     0.03  Kernel#define_singleton_method
  2.30   335.58      1.50    19727     0.08     0.18  BinData::Struct#define_field_accessors_for
  2.23   337.04      1.46    88970     0.02     0.08  BinData::BasePrimitive#_value
  2.17   338.46      1.42    47305     0.03     0.06  Kernel#clone
  2.14   339.86      1.40    89151     0.02     0.06  BasicObject#!=
  2.05   341.20      1.34    31153     0.04     3.85  Array#each
  2.02   342.52      1.32    45817     0.03     0.30  BinData::Base#new
  1.87   343.74      1.22    49057     0.02     0.03  BinData::SanitizedField#name_as_sym
```

結果より、入出力処理やスレッド処理、カーネルに関する処理などの処理に時間がかかっていることがわかるが、これは前節で述べた通り、profileライブラリ自体のオーバーヘッドによるものである(今回のプログラムでは、入出力処理は起動時の一回のみ、スレッド処理は未使用である)。

そのため、実際のCbenchのボトルネックとして考えられるのは上位6位以降のメソッドである。
その多くはオブジェクトのインスタンス生成に関するものや、型に関するものであることがわかる。
cbench.rbのコードに含まれるメソッドとしては、newメソッドがボトルネックとしてあげられる。
実際に、cbench.rbでは、FlomModメッセージを生成する際、
マッチフィールドやアクションを指定するオプションをnewメソッドを用いて生成している。

###発展課題
前節の解析結果より、newメソッドの実行回数を減らすことを考える。
Cbenchでは、コントローラが受け取るメッセージの送信元Macアドレスなどの様々な属性は異なるが、
応答するFlowModメッセージはどのPacket-Inに対しても同じである。
この特徴を利用し、FlowModで送信するメッセージの再利用を行う。

具体的には、以下のようにpacket_inハンドラを変更した。
なお、本変更は[テキスト](http://yasuhito.github.io/trema-book/#cbench)を参考にしている。

```
  def packet_in(datapath_id, message)
    @flow_mod ||= FlowMod.new(
      command: :add,
      priority: 0,
      transaction_id: 0,
      idle_timeout: 0,
      hard_timeout: 0,
      buffer_id: message.buffer_id,
      match: ExactMatch.new(message),
      actions: SendOutPort.new(message.in_port + 1)
    ) 
    send_message datapath_id, @flow_mod
  end
```

この変更により、オブジェクト変数flow_modに値が入っていない場合（一回目のpacket_inの場合）にのみFlowModのメッセージが新たに生成され、
値がすでに入っている場合にはFlowModのメッセージを再利用する。

以下は変更前と変更後それぞれのベンチマーク結果である。
```
[変更前]
RESULT: 1 switches 9 tests min/max/avg/stdev = 63.10/81.99/69.33/4.94 responses/s

[変更後]
RESULT: 1 switches 9 tests min/max/avg/stdev = 281.21/420.95/310.13/42.40 responses/s
```
平均スコアをもとにすると、4倍以上の性能向上が見られる。

####発展課題ファイルへのリンク
[@cbench_accelerated.rb](https://github.com/handai-trema/cbench-shuya-abe/blob/master/lib/cbench_accelerated.rb)

