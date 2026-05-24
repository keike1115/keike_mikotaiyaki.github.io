
# 開発日誌Ver002 light.vn最新版更新と画面領域について

みなさまこんにちわ　keikeです。
本日は26年5月24日になります
今日はlight.vnのバージョンを最新版のVer17にしました。
それに伴うバグ修正をしたのですが、今回、画面領域とアイテム取得UI周りでなかなか面白い設計問題にぶつかったので、備忘録も兼ねてまとめます。

### 前回の復習
壱　アイテム合成こよBoxの作成とデバッグ
弐　それに合わせたgame_item.txtのアイテム処理の改良…内部処理とインベントリUI処理の切り離し
参　lightvnは現在ver16ですので、ver17への更新とデバッグも必要ですね
肆　textbox周りの変更の反映も必要です

今回の更新は主に弐,参の項になります。
light.vn　ver17では主に画面領域に関する更新が行われました。UIが所属する画面領域による作画するしないがはっきり強化されました。

------------------------------------------------------------------------

# 1. 画面領域ってなぁ～に？？

専門用語がでてきましたね。ここでいったん解説を挟みたいと思います。
皆さんはゲームをするうえで、メニュー画面を開いたり、あるいはストーリー演出のためにそれ専用のUIが出てくる経験があったと思います。
<br>・・・さすがにありますよね？？？(汗)<br>
<img src="https://raw.githubusercontent.com/keike1115/keike_mikotaiyaki.github.io/main/images/demo_004.png" width="400"><br>
ここで、それらの演出中にいままでの画像UIやボタンUIが残っていたり、誤って反応してしまったら画面がぐちゃぐちゃになります。
そこで登場するのが画面領域です


``` lightvn
栞　テストA
{
//ケース①
//通常時のゲームボタンやUIを宣言している
絵　objA alice.png 150 30 50
ボタン　objB image.png 0 0 0 スクリプト　ボタン先
画面領域開始　sc_story原点回帰時解除
絵　objC alice.png 150 30 50
M_宣言文字窓その他吹き出し true "みこ" "みこ"　"void"　"#fe4b74" 100 200 100 90
"新しいUIや会話をここで設定するにぇ！\w .ここでは外の階層、通常時UIにあくせすできないにぇ ~
//正常動作としてstoryや新ギミックUI中に画面領域を設定→ボタンBが反応しなくなる、　移動objAが反応しなくなる。便利
//終了時 アウト　.*
//.*は本当はすべて削除。しかし階層領域外のobjAやobjBは無傷　便利
画面領域終了　sc_story
}
//通常基底領域にもどる
```

できあがった画面領域は階層とよばれ、その名の通りパイ生地のように現在の画面領域を終了するまで上に上に積みあがっていきます。
下位階層に「蓋をするように」除外操作をかけるときに役立ちます。使い終わったらその層のobj全消去と、画面領域終了を忘れないように。

とても便利ですが、いくつかのルールや問題を抱えて居ました。


------------------------------------------------------------------------

# 2. 「ストーリー中にアイテムを渡したい」

ゲームを作っていると、

> 会話演出の途中でアイテムを入手する

という場面はよくあります。

たとえばこんな流れ。

コード例：理想の流れ
``` lightvn
みこ「君にアイテムをあげるにぇ！」
↓
アイテム取得演出
↓
会話続行
```

一見、簡単そうに見えます。

しかしここで **画面領域** が絡むと話が変わりました。

自作の `get_item_msg` は、もともと **呼び出すたびにその画面領域でUIを生成する**
設計でした。
手持ちのアイテムがゼロになればアイテム画像UIも消去されます

普段はこれで問題ありません。

しかしストーリー演出では、

``` lightvn
画面領域開始 sc_story
```

を使用しています。

つまり、

-   ストーリーUI → `sc_story`
-   アイテムUI → 基底

という領域分離状態です。

ここでそのまま `get_item_msg` を呼ぶと......

``` lightvn
画面領域開始 sc_story

M_宣言文字窓その他吹き出し ...
"君にアイテムをあげるにぇ！！"

スクリプト game_item.txt get_item_msg
//ここでストーリー演出上の画面領域でアイテムUIが更新、再生成されてしまう
アウト .*
//アイテムが巻き添えで消える
```

```{=html}
</details>
```
後続の `アウト` が期待通り動かず、特にアイテムインベントリUIが巻き込まれました。

つまり、

> 「演出領域」と「基底UI」が衝突していた

わけです。

------------------------------------------------------------------------

# 3. 第一段階：領域を一旦閉じる

最初に思いついた解決策は単純でした。

> アイテム処理中だけ領域を解除する

です。

コード例：最初の回避策
``` lightvn
画面領域終了 sc_story
スクリプト game_item.txt get_item_msg
画面領域開始 sc_story 原点回帰時解除
```

これで確かに動きました。

一件落着......だと思っていたのですが…

------------------------------------------------------------------------

# 4. 第二の壁：分岐で崩壊した

問題は、ストーリーが分岐し始めたときです。

上位スクリプト：

``` lightvn
画面領域開始 sc_story

もし(フラグ == 選択A) スクリプト ルートA
もし(フラグ == 選択B) スクリプト ルートB

画面領域終了 sc_story
```

ルートB：

コード例：下位側
``` lightvn
栞　ルートA
{
    
    //ストーリー演出中にアイテムを渡したいときはあるよね・・・
    M_宣言文字窓その他吹き出し true "みこ" "みこ"　"void"　"#fe4b74" 100 200 100 90
    "アイテムまだ上げないにぇ　出直して。\w
    ~
    スクリプト終了
    
}

待機 続行禁止
栞　ルートB
{
    
    
    M_宣言文字窓その他吹き出し true "みこ" "みこ"　"void"　"#fe4b74" 100 200 100 90
    "君にアイテムをあげるにぇ！！\w
    ~
    画面領域終了　sc_story　
    //OK　下位スクリプト先で終了は許される
    スクリプト　game_item.txt　get_item_msg  "最小空箱" "maguma"　"まぐま"  
    画面領域開始　sc_story　原点回帰時解除
    //まぁOK
      
    //エラー！　画面領域終了を上位スクリプトに投げることはできない。ここで 画面領域終了　sc_story　が必要だが・・・仕様と衝突する
    スクリプト終了
    
}

待機 続行禁止
```

**領域の開始・終了責任が上下スクリプトで分裂した** のです。
上記にもありますが、画面領域終了を上位スクリプトに投げることはできないという問題が発覚しました

結果、

-   どこで閉じる？
-   どこが責任者？

が曖昧になり、仕様と衝突。

これは単なる応急処置では解決しないタイプの問題でした。

------------------------------------------------------------------------

# 5. 一度考えた案

次に考えたのは、

-   ダミーUIを出す
-   後で本処理
-   領域判定で切替

という案。

設計としては綺麗でした。

しかし、

-   API分裂
-   既存コード互換性低下
-   修正範囲増加

という問題もありました。
またこれには現在の画面領域がどの位置にいるのか？ということをif文で判定する必要が有るのですが、
その状態を検知する関数はlight.vnにはありません。

さて困った

------------------------------------------------------------------------

# 6. 発想転換：「領域を変える」必要ある？

ここで考え方を変えました。

問題は、

> どの領域で生成するか

を気にしていたこと。

なら逆に、

> 最初から全部基底で生成すれば？

という発想です。


最終的に採用したのは、

-   生成 → 基底
-   操作 → 基底
-   表示制御 → 透明度

への統一でした。

旧スクリプト
```
//---------------------------------------------------------------
////内部処理(メッセージ挟まず直接与える時はこちら)(初期装備など)
//---------------------------------------------------------------
    
栞　get_item　_inventoryNo　_itempass　_itemname
{
    
   
   //個数チェックとその更新
   //もし ( inventoryname[{{_inventoryNo}}]  == _itemname) スクリプト　game_item.txt　アイテム個数プラス
   //違ったら　スクリプト　game_item.txt　アイテム個数壱
   もし ( inventoryname[{{_inventoryNo}}]  == _itemname) 臨時全域変数　inventoryQTE[{{_inventoryNo}}]　+= 1
   違ったら　臨時全域変数　inventoryQTE[{{_inventoryNo}}]　= 1
   
    
    臨時全域変数　inventorypass[{{_inventoryNo}}]　= _itempass
    臨時全域変数　inventoryname[{{_inventoryNo}}]  = _itemname
    
    変数　tempQTE = inventoryQTE[{{_inventoryNo}}]
    //毎回ボタンを定義していた
    タッチ素材設定　item/item_{{_itempass}}.png  item/item_{{_itempass}}.png  item/item_{{_itempass}}.png  
    ボタン0 Btn_inventory{{_inventoryNo}} (130 * (_inventoryNo%2)  + 820 )   ((_inventoryNo/2)*133 -150) (vn_R値_インベントリ +1) スクリプト　game_item.txt　アイテム持ち替えSorF　{{_inventoryNo}}
    タッチ開始時　Btn_inventory{{_inventoryNo}}　スクリプト　game_item.txt　アイテム名表示　　{{_inventoryNo}}
    拡大　Btn_inventory{{_inventoryNo}}　25%
    .イン　 Btn_inventory{{_inventoryNo}} 300
    文字0 mojiQTE_inventory{{_inventoryNo}} (140 * (_inventoryNo%2)  + 1090 )   (133*(_inventoryNo/2)+160)  (vn_R値_インベントリ +20)  r-mplus-1c-m.ttf 40 "×{{tempQTE}} "
   もし ( tempQTE >= 2)  イン mojiQTE_inventory{{_inventoryNo}} 10
   違ったら　透明度　mojiQTE_inventory{{_inventoryNo}}　0 10

}

スクリプト終了
待機 続行禁止

```

新スクリプト
```
//ボタンの配置をします
栞　アイテム初期UI配置　_inventoryNo　
{
    
    
    変数 _itempass　=　inventorypass[{{_inventoryNo}}]
    変数　tempQTE = inventoryQTE[{{_inventoryNo}}]
    
    
    タッチ素材設定　item/item_undefined.png　item/item_undefined.png　item/item_undefined.png
    ボタン0 Btn_inventory{{_inventoryNo}} (130 * (_inventoryNo%2)  + 820 )   ((_inventoryNo/2)*133 -150) (vn_R値_インベントリ +1) スクリプト　game_item.txt　アイテム持ち替えSorF　{{_inventoryNo}}
    タッチ開始時　Btn_inventory{{_inventoryNo}}　スクリプト　game_item.txt　アイテム名表示　　{{_inventoryNo}}
    反応透明度　Btn_inventory{{_inventoryNo}}　250
    拡大　Btn_inventory{{_inventoryNo}}　25%
    
    変数　mojiObj = "QTE_inv{{_inventoryNo}}"
    M_宣言領域耐性文字 (mojiObj)　(140 * (_inventoryNo%2)  + 1080 )   (133*(_inventoryNo/2)+140)  (vn_R値_インベントリ +20) 40
    "×{{tempQTE}}
    ~
    //イン　sc耐性文字_QTE_inv{{_inventoryNo}}//
    
    
    スクリプト終了
    
}
```
```
栞　get_item　_inventoryNo　_itempass　_itemname
{
    
   
   //個数チェックとその更新
   //もし ( inventoryname[{{_inventoryNo}}]  == _itemname) スクリプト　game_item.txt　アイテム個数プラス
   //違ったら　スクリプト　game_item.txt　アイテム個数壱
   もし ( inventoryname[{{_inventoryNo}}]  == _itemname) 臨時全域変数　inventoryQTE[{{_inventoryNo}}]　+= 1
   違ったら　臨時全域変数　inventoryQTE[{{_inventoryNo}}]　= 1
   
    
    臨時全域変数　inventorypass[{{_inventoryNo}}]　= _itempass
    臨時全域変数　inventoryname[{{_inventoryNo}}]  = _itemname
    
    変数　tempQTE = inventoryQTE[{{_inventoryNo}}]
    
    透明度　 Btn_inventory{{_inventoryNo}}　0　全画面領域
    画像　Btn_inventory{{_inventoryNo}}　item/item_{{_itempass}}.png　全画面領域
    
    .イン　 Btn_inventory{{_inventoryNo}} 300　全画面領域
    使用文字窓 sc耐性文字_QTE_inv{{_inventoryNo}}
    "×{{tempQTE}}
    ~
    もし ( tempQTE >= 2)  イン sc耐性文字_QTE_inv{{_inventoryNo}} 10　全画面領域
    違ったら　透明度 sc耐性文字_QTE_inv{{_inventoryNo}}　0 10　全画面領域

}

スクリプト終了
待機 続行禁止

```
つまり、

> 画面領域は演出管理だけ

にする。

アイテムUIは **基底サービス** として扱う。

これにより：

コード例：どこからでも呼べる
``` lightvn
画面領域開始 sc_story
スクリプト game_item.txt get_item_msg

画面領域開始 sc_menu
スクリプト game_item.txt get_item_msg

画面領域開始 sc_battle
スクリプト game_item.txt get_item_msg
```

すべて正常動作。

もう、

終了 → 処理 → 再開始
現在領域判定

どちらも不要です。

------------------------------------------------------------------------

# 8. 結び

今回の件で面白かったのは、最初は「バグ修正」だったのに、最終的には
 UI責務の再設計になっていたことです。

大きな手術と発想の転換が必要でしたが、それでも便利な機能同士ほど、責務分離は大事。
毎回objは消せばいいってもんじゃないんですねぇ～

さて、、今回はここまで。
次回はいよいよこよＢｏｘの完成を目指します。
これで心置きなくストーリー演出中にアイテムを渡せるぞぉ！！
あ、ついでに今までにも使っていた

``` lightvn
画面領域終了 sc_story
スクリプト game_item.txt get_item_msg
画面領域開始 sc_story 原点回帰時解除
```
部分の修正も忘れずにね！！

それではまた次回お会いしいましょう！
