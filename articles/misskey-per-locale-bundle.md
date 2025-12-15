---
title: "Misskey の Per-Locale バンドルのビルドシステムを作った話&技術解説"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['misskey', "vite", "i18n", "vue"]
published: true
---

[Misskey Advent Calendar 2025][calendar] 13日目の記事です。

-----

こんにちは。 [にりらみすきー部][misskey.niri.la] のMisskeyのメンテナで misskey-dev の一員になった anatawa12 です。

Misskey 2025.8.0 ではフロントエンドのバンドルの構造が大きく変更されました。
このバージョン以降の misskey では、フロントエンドのバンドルが言語別に分割され、また書く言語の翻訳情報が javascript 内にインライン化されるようになりました。

この記事では、これの実装方法等の技術的な話をします。
実際に実験を含めて行ったのはPR[#16369]ですので、更に詳しく見たい方はそちらもご覧ください。

## 実装経緯 {#background}
<small>詳しくは[#14453][#14453]を参照できます</small>
Misskey には大量の翻訳対象テキストがあります。
しかしそれらには通常使用では使用しない文字列が含まれています。
例えば、管理者向けの設定画面のテキスト、Misskey Games用のテキスト、設定画面の詳細説明などです。

これらのテキストを含めていると、翻訳ファイルが大きくなってしまいます。
元々はこれのcode splittingを行うことが目的でした。
また、定数にすることで実行時コストを減らすことも期待していました。

また、翻訳ファイルのロードのタイミングが噛み合わないと、古いバージョン向けの翻訳ファイルを参照してしまうことによるエラーも発生していました。

これらの問題を解決するために、翻訳ファイルをバンドルにインライン化し、また言語ごとにバンドルを分割することにしました。

## 実装方法
### First try: viteでコードを分ける

まず最初に試したのは、vite の plugin を書いて、言語ごとにコードを分ける方法です。
viteでは、pluginが存在しないファイルを別のファイルを元に生成することができます。
また、パス名に `?` をつけてクエリパラメータを定義することで、派生ファイルを表現することが(規約の上で)できます。[^vite-query]
これを使用して、`path/to/some/module.vue?locale=ja` のようなファイルをプラグインで生成し、またプラグインで i18n の import とそれへのアクセスを置き換えることでの実装を試みました。

しかし、この方法ではビルド時間が大幅に(手元のマシンで10分超えまで)増加してしまいました。
アタリマエのことですが、言語ごとに全てのコードを複製してビルドすることになるため、ビルド時間が言語数倍になってしまいます。
そして misskey は多くの言語をサポートしているため、ビルド時間が非常に長くなってしまいました。
コア数分で並列することも考えましたが、ビルド時間が長いことには変わりなく、またビルドマシンのコア数に依存してしまうため、あまり良い方法ではありませんでした。

しかし、言語の置き換え自体はそこまで負荷が高くない処理だと考えたため、次の方法を試しました。

### Second try: ビルド後にバンドルを分割する

次に試したのは、通常通りにビルドを行い、その後にバンドルを言語ごとに分割する方法です。
この方法では、まず通常通りにビルドを行い、全ての言語を含むバンドルを生成します。
その後、生成されたバンドルを解析し、言語ごとに必要な部分だけを抽出して新しいバンドルを生成します。

#### Identifying i18n imports
最初からこの方法を試さなかったのは、minify されたコードから元の i18n import 部分を特定するのが難しいのではないかと考えたため、また複数ファイルの bundle によって影響を受けるのではないかと考えたためでした。
しかし、それぞれの問題は以下のようにクリアできました。

minifyされていることに関しては、i18nのexportをminifyさせないように設定できました。
`rollupOptions.input`に i18n.ts の追加と`rollupOptions.preserveEntrySignatures`を`allow-extension`に設定することで、i18n.ts の export 部分が minify されないようにできました。

`rollupOptions.input`は、 rollup のエントリーポイントを追加するためのオプションです。
通常の場合 html に指定したファイルがエントリーポイントになりますが、ライブラリなどのようにエントリーポイントが複数ある場合などに使用することができます。おそらく比較的ポピュラーな設定だと思います。
これを設定することで`manifest.json`にi18n.tsのバンドル後のファイルが記載されるようになりました。

しかしこのオプションを使用するだけではi18nという名称がminifyされてしまいました。、内部コードからの import 文はラップ元のファイルを参照するようになってしまいます。
これでは i18n.ts の import 文の特定には不十分だったため、rollupのドキュメントを片っ端から調べたところ、`rollupOptions.preserveEntrySignatures`というオプションを見つけました。
このオプションは、エントリーポイントのexportをどのように変換するかを設定するものです。
この設定値には `strict` 、 `allow-extension` 、 `exports-only` 、 `false` の4つがあります。

- `strict` の場合、エントリーポイントの export はそのまま厳密に維持され、追加の export が生成されません。
  この動作を実現するため、rollup はバンドルエントリーポイント用のラッパーコードを生成し、その中でバンドルされたファイルを import & export する事があるようです。[^preserve-entry-point-signatures]
- `allow-extension` の場合、エントリーポイントの export は維持されますが、追加の export が生成されることがあります。
  この場合は rollup はエントリーポイント用のラッパーコードを生成せず、バンドルされたファイルがentrypointとして
- `false`の場合は何も維持されず、全ての export が自由に変更されます。
- `exports-only` の場合、エントリーポイントの各ファイルに export があれば `strict` と同様に、なければ `allow-extension` と同様に動作します。

そしてviteのクライアントアプリケーション向けのデフォルトがfalseであることがわかりました。[^vite-preserve-entry-default]
そのため、このオプションを `allow-extension` に設定することで、i18n.ts の export 部分が維持されるようになりました。
これにより、minifyされたコードから i18n の import 部分を特定できるようになりました。

### Preventing i18n from bundling with other files
しかし、このままでは i18n.ts が他のファイルと bundle されてしまい、importを削除できない可能性がありました。
importされてしまうと code splitting の効果が薄れてしまうため、 i18n.ts の依存関係でかつ別のファイルから参照されているファイルを特定し、それらを i18n.ts の bundle から分離する必要がありました。
この問題は、vite の `rollupOptions.manualChunks` オプションを使用して解決しました。
実際に依存関係で問題になったのは config.ts だったため、これを manualChunks に指定することで config.ts を i18n.ts の bundle から分離しました。

これにより、i18n.ts の import 部分を削除してロケールのインライン化により、言語ファイルのロードを防止できるようになりました。

### Process `unref` of vue
この状態で i18n.ts の import もその使用方法も特定できるようになったため意気揚々とインライン化をしようとしたところ、vue の SFC の template 内での i18n の参照がすべて `unknownFunction(i18n)` のようになってしまっていることに気づきました。
viteのminifyを無効化してビルドするなどで調査したところこの関数は `unref` であることがわかりました。
vueの内部ついてそこまで詳しくはなかったのですが、vue の template タグ内ではリアクティブな変数をそのまま参照できるようになっており、そのために `unref` が自動的に挿入されているようです。
しかしi18nは `markRaw` されているリアクティブな変数ではないため、`unref` の意味は存在しなかったためこれを特定して削除する必要がありました。
しかし`unknownFunction`が`unref`ではない場合には削除してしまうと不具合が発生してしまうため、正確に `unref` のみを特定して削除する必要がありました。

これの解決にも `rollupOptions.inputs` を使用する事もできたのですが、そうしてしまうと vue の関数名の長い関数がすべて展開されてしまい、バンドルサイズが大きくなってしまうため、避けたいと考えました。

そのため、最終的には vite の[プラグイン][vite-plugin-remove-unref]を書いて unref(i18n) のみを特定して削除することにしました。 [^vite-plugin-remove-unref]

### Inline locale data

これによって i18n.ts の import 部分とその使用方法を特定できるようになったため、あとは言語ごとに i18n の使用箇所を言語データで置き換え、分割したあとのファイル名にすべての import を変更するだけとなりました。
この処理は複数の言語にたいして複数回行う必要があるため、共通してパースを一回行い、その情報をもとに言語ごとに置き換えを行うようにしました。

パーサーは `i18n` の import 文を特定し、それのminifiedな名前を取得し、その定数によるプロパティーアクセスを特定し、そのソースコードの範囲と翻訳キーのタプルの一覧を取得するようにしました。
またすべてのimport文の書き換えが必要であったため、 文字列リテラルの`scripts/***.js`のscriptsの範囲も同様にパースして取得しました。

パーサーで変数アクセスを収集する際にバグをいくつか発生させていた点として、i18n二アクセスしていないスコープの変数としてimport文と同盟の変数をminifierが生成している事がありました。これの除外がすごく面倒臭かったです。
変数宣言は、スコープがそのノードの親ノードの範囲であるため一旦非対応とし、関数引数になっている場合にのみインライン化の対象外としました。
この処理に未だに非対応が残っている状態なので、将来的に修正するべきだとは考えています。

置き換えは元々はローケルの値が定まっている場合にのみ行おうと考えていたのですが、文字列かどうかの判定を置換前に行うのが少々面倒であったのと、`json.stringify`を使用することで高速に置き換えができたため、すべての置き換えを行うようにしました。
具体的には`i18n.ts.accept`は`"許可"`に、`i18n.ts._timelines[column.tl]`は`({"home":"ホーム","local":"ローカル","social":"ソーシャル","global":"グローバル"})[column.tl]`のように置き換えを行いました。

また、パラメータ付き翻訳定数については、misskeyのi18nは関数呼び出しでパラメータを受け取るため、即時実行のラムダ式にすることで対応しました。
具体的には`i18n.tsx._dialog.charactersExceeded({ current, max })`は`((({current,max})=>("最大文字数を超えています！ 現在 "+current+" / 制限 "+max))({ current, max })`のように置き換えを行いました。

### Disable prefetching i18n.ts
これでインライン化完成と思って実際にビルドして実行してみると、何故か i18n.ts (のコンパイル後のファイル)がフェッチされていることに気づきました。
調査したところ vite はdynamic import文を置き換えて prefetch する仕組みがあるようで、これによって i18n.ts がプリフェッチされてしまっていました。
prefetch自体は効いてくれたほうが良いのですが、i18n.ts の prefetch だけ選択的に無効化する必要がありました。

minifyされているコードとにらめっこしてたのですが、きれいにprefetchを削除するのは難しそうでした。そのため最終的に i18n.ts を prefetch しようとした際には自分自身を prefetch しようとするようにパスのテーブルを書き換えることで、むだな prefetch を防止しました。

### Final touch
これらを実装した後に、ビルド時間が大幅に増加していないことを確認し、また生成されたバンドルが正しく動作することを確認し、最終的にロケールのロード処理を削除する変更を加えて完成となりました。

今までの処理はfrontendを対象としていたのですが、 misskey では embed 用のバンドルは別途存在するため、同様の更新を embed 用のバンドルにも適用しました。

最後に、もともといくつかのjsonがあることに依存していた処理を書き換えることで、完全に翻訳ファイルのロードを不要にしました。

## まとめ
以上が Misskey の言語別バンドルの実装方法の解説および技術的な話でした。

もっと聞きたい所があれば書いてくれると追記しますので声かけてください

[calendar]: https://adventar.org/calendars/11291
[misskey-readme]: https://github.com/misskey-dev/misskey#readme
[misskey.niri.la]: https://misskey.niri.la/#pswp
[Vmimi Relay Timeline Fork]: https://github.com/anatawa12/misskey/tree/vmimi-relay-timeline/releases/?tab=readme-ov-file#vmimi-relay-timeline
[Vmimi Relay]: https://relay.virtualkemomimi.net/
[#14448]: https://github.com/misskey-dev/misskey/pull/14448
[#14453]: https://github.com/misskey-dev/misskey/issues/14453
[#14453]: https://github.com/misskey-dev/misskey/issues/14453
[#16369]: https://github.com/misskey-dev/misskey/pull/16369
[vite-plugin-remove-unref]: https://github.com/misskey-dev/misskey/blob/9e1e40d35a638c3383f8b6ba5b4a730a48270eb1/packages/frontend-builder/rollup-plugin-remove-unref-i18n.ts


[^vite-query]: 詳しく理解してるわけではないですが、vite/rollup的にはファイルの区別はパス名の文字列でしか行っておらず、拡張子の確定に?以降を無視するという規約がある模様です。
[^preserve-entry-point-signatures]: 今試したところ下記の config.ts を分離したあとにだと strict でも直接 import されたので、exportsが増えない場合にはラッパーコードは生成されないようです。今回の用法では `exports-only` にしても良かったのかもしれないです
[^vite-preserve-entry-default]: https://github.com/vitejs/vite/blob/977d9ee28df7b2a95db5ec3b1fde27033c485199/packages/vite/src/node/build.ts#L621
[^vite-plugin-remove-unref]: このプラグイン、viteの公式プラグインと違って dev モードでもパースしちゃってるので開発環境でちょっと重めかも。後処理用のものなので開発環境ではこのプラグイン自体無効化してもいいですね。
