---
title: "ポケモンSVのグリッチを自動化する"
date: 2022-12-23T21:14:27+09:00
draft: false
---
この記事は [TDU Advbent Calendar 2022 11日目](https://qiita.com/advent-calendar/2022/tdu)の記事です。

「ポケットモンスター スカーレット・バイオレット」には現時点（v1.1.0）でポケモンに持たせられるアイテムを増殖させるグリッチ、通称:増殖バグが存在します。\
もちろんBANされる可能性はありますし、そもそも倫理的によろしくない行為であることは間違いないので使用は自己責任になります。\
手動でやると正直面倒なのでマクロ機能を持つコントローラーで自動化されてる方もいらっしゃいますが、Switchは自動化の情報がそこそこ出回っているので自前で自動化することも可能です。\
基本はArduinoなどのマイコンを使うことが多いのですが、今回は用途を失って部屋に転がっていた[ラズパイ3B](https://akizukidenshi.com/catalog/g/gM-11425/)で自動化してみようと思いました。

## ラズパイの準備
まずはラズパイ経由でSwitchを操作できるようにする必要がありますね。

[Arduino を使わずに Bluetooth 経由で Nintendo Switch の操作を自動化する【ポケモン自動化】【v13.0.0 まで対応】](https://qiita.com/almtr/items/38a7f0c3056024532e8d)\
[Raspberry Pi で Nintendo Switch を自動化！単純作業を効率化しちゃおう](https://ponkichi.blog/rapsberry-switch-01/)


基本的なことは偉大な先人たちの記事にかなり詳細にまとめられているので、手順通りやれば問題ないかと思います。\
何点か躓きそうなポイントがあるので記載します。

### ペアリング
基本的には問題ないのですが、たまにラズパイ側は認識しているのに、python側が認識していないケースが発生します。\
PairingController.py実行後の
```
[00:00:00] joycontrol.server create_hid_server::96 INFO - Waiting for Switch to connect… Please open the "Change Grip/Order" menu.
```
のログが出た直後のタイミングでSwitchのコントローラー再接続画面を開くと比較的安定してペアリングすることが出来ました。\
接続が失敗していたらbluetoothctlでSwitchのMACアドレスをremoveしておきましょう。

### ペアリング後のコントローラー操作
自動化テスト中などに自分でコントローラー経由の操作をしたくなるタイミングがあると思いますが、ペアリングさえしてしまえばJOY-CONやプロコンを有線で繋いで操作を加えても問題なくスクリプトが走ります。

### 動作の安定化
Bluetooth経由なので時々遅延や抜けが発生したり、waitが短すぎると反応しなかったりします。\
あとは干渉しないような環境を整えたり、switch本体とラズパイを離しすぎないのもポイントかと。\
ついでにポケモンSV側の動作がやや不安定なところもあるので、できるだけ処理が軽くなるようにしておくと安定しやすいと思います。
- フィールドマップは天候変化やポケモンの湧き処理が走るので、寮の自室など処理が軽めのところへ移動する
- ボックスは大量にポケモンがいると重くなるので、逃したり、サブロムに移したりして軽くしておく
- タイミングを見計らってSwitch本体を再起動する

など

## 自動化の実装
とりあえずテキトーに書いて動作テストしたものが以下になります。
{{< rawhtml >}}
<iframe src="https://drive.google.com/file/d/1DvF-GAWHd6YjqkP7TYZHkLHwCfYhTNzp/preview" width="640" height="480" allow="autoplay"></iframe>
{{< /rawhtml >}}
「かがくのちからってすげー！」

## スクリプトを書く
このままだと一回あたりの増殖に時間がかかりすぎなので効率化します。\
「カーソルが手持ち先頭のバグイドンに合わせてある」かつ「手持ちが2体」状態からだと以下のコマンドで一周することになります。
- バグイドンをライドフォルムに戻す\
A ↑ ↑ A A A A
- ボックスを開く\
↑ → A
- 増えたアイテムを回収\
X X L A ↑ ↑ ↑ A
- ボックスを閉じてカーソルを合わせる\
B ← ↑

というわけでサンプルコードを参考に処理を書いていきます。

```py
import logging
from JoycontrolPlugin import JoycontrolPlugin

logger = logging.getLogger(__name__)

class MultiplicationGlitch(JoycontrolPlugin):
    async def run(self):
        logger.info('processing start')
        processingCount = 999
        count = 0
        while count < processingCount:
            await MultiplicationGlitch.Multiplication(self)
            count += 1
        logger.info('done')

    async def Multiplication(self):
        shortWait = 0.15
        middleWait = 1
        longWait = 2
        screenWait = 2.5
        commands = [
            {'button': 'a', 'wait': shortWait},
            {'button': 'up', 'wait': shortWait},
            {'button': 'up', 'wait': shortWait},
            {'button': 'a', 'wait': longWait},
            {'button': 'a', 'wait': middleWait},
            {'button': 'a', 'wait': longWait},
            {'button': 'a', 'wait': middleWait},
            {'button': 'up', 'wait': shortWait},
            {'button': 'right', 'wait': shortWait},
            {'button': 'a', 'wait': screenWait},
            {'button': 'x', 'wait': shortWait},
            {'button': 'x', 'wait': shortWait},
            {'button': 'l', 'wait': shortWait},
            {'button': 'a', 'wait': middleWait},
            {'button': 'up', 'wait': shortWait},
            {'button': 'up', 'wait': shortWait},
            {'button': 'up', 'wait': shortWait},
            {'button': 'a', 'wait': middleWait},
            {'button': 'b', 'wait': screenWait},
            {'button': 'left', 'wait': shortWait},
            {'button': 'up', 'wait': shortWait},
        ]
        for command in commands:
            await self.button_push(command['button'])
            await self.wait(command['wait'])
```
ロードやメッセージ表示が入るところや処理落ちがかかるところは長めのwaitを入れてあります。\
大体1個あたり16秒くらいで増殖できるようになりました。

## 結果
テラスピースのオシャボも使い放題や！\
とはなりませんでした、というのも実行時のコマンド抜けやポケモン側の想定外の処理落ちなどがそこそこの頻度で発生するのでどこかで処理をミスると復帰できないのです。\
メニュー閉じたりアイテムを持たせ直したりするロジックを組み込んでは見たものの、それでも調子が悪い時は動作が不安定になります。\
ただ、手動で増やすよりは何倍も楽なので別ゲーをやりつつ再就職したラズパイを眺める年末になりそうです。\
皆様もお手元で眠っているラズパイ、再就職させてゲーミングライフを快適にしませんか？
