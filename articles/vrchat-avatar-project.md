---
title: "anatawa12のVRChatアバタープロジェクトを紹介する"
emoji: "🐱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vrchat"]
published: true
---

Twitterで需要ある？って聞いたらあると言われたので私のアバタープロジェクトの構成を紹介してみます。
AvatarOptimizerのテスト環境と共通です。

:::message
あくまでも個人的に使用してる形式です。また推奨できない部分があります。
:::

:::message
知りたいことがあれば [twitter](https://twitter.com/anatawa12_vrc) に聞いてください
:::

### プロジェクト数は
答え: 使用してるのは 1 project。すべての手持ちアバターを同じプロジェクトにまとめてます。VCC検証用に小さいプロジェクトは結構な数ありますが無視します。
理由: Avatar Optimizer等の開発の都合があったりプロジェクトの開き直しが面倒なため。

### シーン数は
答え: 1アバター 1 シーンになってます。
理由: 複数アバター 1 シーンは見づらくなったりする気がするので。

### 差分管理は
答え: 多重prefab variantを使用してます。私の場合メガネ/髪飾りvariantを作り、そのvariantで服着せてます。
理由: まぁ一番素直な方法だと思うので。

### バックアップ方法
答え: ローカルにgit + [git-vrc]、リモートにgithub private repositoryです。LFS[^lfs]はつかってません。VCCの機能は使ってないです。
理由: commitで変更点を追うのが大変便利なので。tagつけることでかんたんに前回のバージョンにロールバックできますし。 また、git branchでquest版を作ってます。

[git-vrc]: https://github.com/anatawa12/git-vrc

### 環境は
答え1: 主には最新版のmacOSです。たまにwindows 11も使いますが 99% macOS です。
理由1: macOS、POSIXでgitとか使いやすいしPOSIX sh (zsh)(とcommandキーの組み合わせ)が最高なので。

:::message alert
macOS上で開発するのは意外と問題なく開発できますが、おすすめはしません。
ノウハウ聞かれたら無限に答えますが。
:::

答え2: VCCは公式のを使わずに vrc-get という自作のやつ使ってます。
理由2: VCCのmac用コマンドあるんですが機能不足なので vrc-get 使ってます。

答え3: テキストエディタ / IDEはRiderを使用してます。
理由3: JetBrains信者なんです。VSCodeは補完弱いし使い慣れてない。Visual Studio使い慣れてない。

### プロジェクトはどこおいてる

答え1: macOSでは普通に`~/UnityProjects`以下においてます
理由1: 特にないです。

答え2: windowsではへのsymlinkを`%USERPROFILE%\UnityProjects`から`D:\UnityProjects`に貼ってます。
理由2: Dドライブに押し出したかったんですがRider等の設定変えたくなくてsymlinkにしました。ジャンクションは普通に存在を知りませんでした。Windowsはまじでわかってない人間です

### VCCっていつから使ってる？
答え1: 2022年5月18日にVCC beta (git VCC)を使用開始した記録があります。
答え2: 2022年7月23日に現行形式のVCC(vpm repo + zip形式)に移行した記録があります。
理由: unitypackageで管理したくないと思っててupm学び始めた時期にこんなの降ってきたら使わない理由ないじゃんって速攻飛びつきました。[当時のツイート](https://twitter.com/anatawa12_vrc/status/1526740641409269765?s=20)

:::details どうでもいい話

- ちなみにこの情報もgit管理から見つけたデータだったりします
- 最初のVCCはgit upmに依存してたんですよね〜 VRCSDK 3.0.x時代懐かしい。
- ちなみにVCC最初期当時のSDKは[ここ](https://github.com/vrchat/packages/tree/refs/tags/3.0.4)に残ってたりします。
- いまみてみたらVCC (Creator Toolbox) 発表当日なんですね VCC beta使用開始
- ちなみにVPM VCCへの移行は少し遅れた記憶があります。以降が発表された当時unityから離れてた気がする。

:::

## 使用してるパッケージ

### VCC/VPM経由

#### com.vrchat.core.vpm-resolver
VPM/VCCの基礎機能の一つ。更新等は経由してますがgitレポジトリにあるのでVCC経由と言えるかは微妙です。
残りのVCC経由のやつはこれ経由で入れてくれます。
vrc-get resolverができたら([issue](https://github.com/anatawa12/vrc-get/issues/86))そっちに移行したいなぁ...

#### com.vrchat.base
単なるVRCSDKの前提のはずなのになぜか依存として定義されてますね。VCC初期のバグの類…?

#### com.vrchat.avatars
VRCSDKなので当然いれてます。

#### [nadena.dev.modular-avatar](https://github.com/bdunderscore/modular-avatar)
皆さんご存知のModular Avatar。私のアバタープロジェクトの最も基礎的で重要なツール。壊れたら私のアバタープロジェクトが壊れます

#### [com.anatawa12.animator-controller-as-a-code](https://github.com/anatawa12/AnimatorController-as-a-Code)
アニメータコントローラをGUIじゃなくてスクリプトで作りたいなと思ってた時期に作ったツールなんですが結構放置しちゃってます。

#### [vrchat.blackstartx.gesture-manager](https://github.com/BlackStartx/VRC-Gesture-Manager)
下に書くAv3Emulatorの前提です。Av3EmulatorがVCC管理下ではないのでこっちに書いといてます。

#### [jp.lilxyzw.liltoon](https://github.com/lilxyzw/lilToon)
皆さんご存知のliltoon。もともとはgit upmでしたがvpmに移行しました。

#### [com.anatawa12.gists](https://github.com/anatawa12/unity-gist-pack/)
私のgistに上げたちっちゃいutilitiesをvpmにまとめたやつ。
CompileLogger, PhysBoneEditorUtilities, SetRandomBlueprintId, CompilationLogWindow, ActualPerformanceWindow の5つが有効化されてます。ActualPerformanceWindowはAvatar Optimizer使ってるとほぼ必須級のやつです

#### [jp.lilxyzw.avatar-utils](https://github.com/lilxyzw/lilAvatarUtils/)
少し前に話題になったlilさんのやつ。だけどまだほぼ使ってないですがいれてます。話題になってから入れてからずーっとほとんどAvatar Optimizerの作業やってるのでw

#### [de.thryrallo.vrc.avatar-performance-tools](https://github.com/Thryrallo/VRC-Avatar-Performance-Tools)
テクスチャごとのVRAM使用量を可視化してくれるやつ。 どのテクスチャがVRAM余計に食ってるか見れるので、VRAMがperformance rankにはいったときにこれを使って容量減らすテクスチャを探しました。

### git submodule経由

:::details git submoduleとは
端的に言うとプロジェクトの中に別のgitプロジェクトを突っ込めるやつです。
vpmになってなかったり開発中だったりするやつを突っ込んでます。
:::

#### [com.anatawa12.avatar-optimizer](https://github.com/anatawa12/AvatarOptimizer)
私を有名にしたAvatarOptimizerは開発中なので vpm レポジトリを経由せずにここにあります。
リリースされてない最新版を使ってる状態。

#### [com.anatawa12.continuous-avatar-uploader](https://github.com/anatawa12/ContinuousAvatarUploader)
1アバター1衣装向けの連続アップロードツールを作ろうとレポジトリだけ作ったやつです。AvatarOptimizerと同様に開発用にここにあります。

**まだ何も機能がありません** なのに star が 10 も付いてるんですよね... 期待されちゃってる

#### [jp.lilxyzw.scene-view-extensions](https://github.com/lilxyzw/lilSceneViewExtensions/)
メッシュの法線とかを見られるツール。1.0までvpm repoに入れないと言ってましたのでgitにいれてます。
AvatarOptimizerのバグ調査で使えそうだと思って使わせてもらってます。

#### [lyuma.av3emulator](https://github.com/lyuma/Av3Emulator)

GestureManagerより高度なAvatars 3.0のunity上のエミュレータです。
curatedにはいるならそっちで、入らなければ独自のレポジトリ建てるそうなので、方針定まるまではsubmodule運用してます。

### その他

#### [com.anatawa12.editor-extension](https://github.com/anatawa12/VRC-Unity-extension.git)

メンテしてないBoneFixerがあるやつですが、多分今後gist pack行きしますね。

[^lfs]: [git LFS](https://github.com/git-lfs/git-lfs)はgitの拡張機能の一つです。
