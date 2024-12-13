---
title: "独自タイムライン VRTL を作った話とそれに伴って misskey の timeline 処理をリファクタした話"
emoji: "📘"
type: "tech"
topics: ['misskey']
published: false
---

[Misskey Advent Calendar 2024][calender] 13日目の記事です。

-----

こんにちは。 [にりらみすきー部][misskey.niri.la] のMisskeyのメンテナで [Vmimi Relay Timeline Fork] を作ってて misskey-dev の一員になった anatawa12 です。

Vmimi-Relay Timeline (VRTL) というものを Misskey に追加した話、それの実装方法、それに伴って Misskey の timeline 処理をリファクタした話をします。

## Vmimi-Relay Timeline って何

Vmimi-Relay Timeline は、ぶいみみリレーからのノートを表示するためのタイムラインです。

ぶいみみリレーはVRChatとかResoniteとかClusterとかのユーザを想定した Activity Pub サーバーのリレーサーバーです。
VRChatとかResoniteとかClusterとかは VR SNS 、いわゆるメタバースと呼ばれるものの一種になります。

> VRChatとかResoniteとかClusterとかそのあたりのユーザーを想定した小規模サーバー向けのリレーサーバーです。
([Vmimi Relay]ホームページから引用)

なので、Vmimi-Relay Timeline は VR SNS 関連サーバーからのノートを表示するためのタイムラインです。

実際にはぶいみみリレーからの投稿のみを表示する Vmimi Relay Timleine (VRTL) と、 ぶいみみリレーまたはフォロー中のユーザーの投稿を表示する Vmimi Social Timeline (VSTL) の２つのタイムラインを作成しました。

## 作った経緯

私の参加してるにりらみすきー部にて、ぶいみみリレーからのノートを表示するためのタイムラインが欲しいという話が出ました。

理由としては、

- Local Timeline だと VR SNS (VRChat) の話が追いきれない
- Global Timeline だと Misskey.io などの他のインスタンスの話が多くて VR SNS の話が埋もれる

というのがありました。

この解決として、ぶいみみリレーからのノートを表示するためのタイムラインがほしいよねということで作ることになりました。

## 実装について

### サーバー/backend側

:::details 本当の最初の実装

本当の最初には、 Vmimi Relay Timeline のみを作成しており、その際には Global Timeline をコピーしていたため、 DBへのアクセスでした。

直後に Vmimi Social Timeline もほしいよねということになり、 VSTL を作成する際に FTT[^FTT] ベースの実装に置き換えたため、この記事ではこっちの実装を扱います。

:::

まずはじめにノートのフィルタリング方法について考えました。
私はActivity Pub Relayの仕組みを理解してなかったので、リレー経由と直接のアクティビティが区別ができないと思ってました。
そのため、リレーからのノートかの区別には参加サーバーの一覧を持っておいて、そのサーバーからのノートかどうかで判定することにしました。

https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/backend/src/core/VmimiRelayTimelineService.ts#L65-L71

参加サーバーの一覧については、 ぶいみみリレーが[参加サーバーを JSON で公開してくれている][vrtl-servers-api]ので、それを定期的に取得する実装になっています。

次に、タイムラインの実装です。
現在、 Misskey では Fanout Timeline Technology[^FTT] が実装されています。 これはタイムラインに載るノートのidを Redis に載せておき、それを取得することでタイムラインを表示するものです。

新規のタイムラインを作成するには、まず Redis 上のタイムラインを追加する必要があります。`FanoutTimelineService.ts` にて、タイムラインの名称が定義されているため、そこに追加します。

https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/backend/src/core/FanoutTimelineService.ts#L41-L44

次に、Redisへノートを追加します。追加処理は `postNoteCreated` 内で条件マッチし、条件に適合したものを Redis に追加するだけです。
Redisへの追加は pipeline を使用してまとめて処理してるので、それに乗っかりましょう。

https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/backend/src/core/NoteCreateService.ts#L978-L983

エンドポイントについては、FTTを使用するタイムラインエンドポイント用のServiceがあるので、それを使用している Social / Local Timeline の実装をコピペして統合対象のRedis Timelineを変更することで主な実装が終わります。
必要であれば DB Fallback のほうも再実装して上げる必要があります。

https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/backend/src/server/api/endpoints/notes/vmimi-relay-timeline.ts#L112-L133
https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/backend/src/server/api/endpoints/notes/vmimi-relay-hybrid-timeline.ts#L120-L155

なお、VSTLは統合対象の Redis Timeline が多いため、遡る際に [ノートの取得漏れが起きる][dev-issue-13488] 問題が発生したため、にりらみすきー部では後に [その改善PR][dev-pr-13495] を cherry-pick することになりました。

Streaming API は LTL STL などをコピペして Streaming チャンネルを新設し、フィルタを書き換えることで実装しました。

https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/backend/src/server/api/stream/channels/vmimi-relay-timeline.ts#L66-L67
https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/backend/src/server/api/stream/channels/vmimi-relay-hybrid-timeline.ts#L58-L67

### クライアント/frontend側

クライアント側では、 STL HTL GTL LTL に続くタイムライン種別として、 VRTL VSTL というタイムラインを追加しました。

まず初めに、タイムラインの描画を担当する `MkTimeline.vue` に、StreamingとPagingのエンドポイントの定義を追加します。

https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/frontend/src/components/MkTimeline.vue#L124-L135
https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/frontend/src/components/MkTimeline.vue#L208-L221

次に、各種タイムラインのページで、VRTL VSTL への切り替えと、 withRelies の有無を調整します。
当時の Misskey では STL HTL GTL LTL それぞれの実装は Timeline Widget, Deck UI, Default UI のそれぞれで完全に分離していたため、それぞれ個別に条件分岐を書く必要がありました。
(Timeline Widget については私が機能を認知していなかったために、最初のリリースの際には Timeline Widget には対応を追加し忘れていました。)

https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/frontend/src/pages/timeline.vue#L309-L318

https://github.com/anatawa12/misskey/blob/81dad3f00e8acc8138a4d63bfee01d6103dd116d/packages/frontend/src/ui/deck/tl-column.vue#L90-L105

私はこの分散をあまり良く思わなかったため、実装後にリファクタリングを行い、これを改善することにしました。

## タイムラインのフロントエンドのリファクタリング

前述の通り、クライアント側のタイムラインに関する情報が各タイムラインの描画の実装に分散していたため、タイムラインの追加に苦労しました。
そのためこれを改善するリファクタリングを行い、本家に取り込んでもらうこととしました。

とりあえず直ぐにできるであろう範囲として、以下の情報を一箇所で定義するようにしました。これはコピペコードを一箇所にまとめるだけとなるので、比較的通りやすいだろうと考えたためです。

- タイムラインの種別の名前 (union型と一覧)
- ロールやインスタンスポリシーに基づくタイムラインの有無
- withReplies のようなオプションの有無
- タイムラインの種別に対するアイコン

このようにすることで、新しいタイムラインを追加する際には、ここを変更するだけで済むようになります。
また、描画ファイル内に散らばっていた長大な三項演算子や論理式がswitchなどを用いて見やすい形になったと思います。

https://github.com/misskey-dev/misskey/blob/e8bf6285cb023bd2031e85de3a3eb495bf4afb2e/packages/frontend/src/timelines.ts

また、このPRを取り込んだ際後のVRTLの実装は主にはこのファイルと`MkTimeline.vue`を変更することになり、比較的シンプルになりました。

## 今後について

VRTL VSTL の実装はほぼ問題なく動いてますが、リファクタリングについてはサードパーティタイムラインを想定すると以下の課題が残ってると思っています。

- 追加タイムライン固有のフラグが表現できない
  - 現在のVRTL VSTLは `withLocalOnly` という連合なしノートを含むかのフラグを持っていますが、これはそれぞれのファイルでフラグ自体は追加しなければなりません。
- フラグの排他性が表現できない
  - 現在 withFiles withReplies は排他オプションとなってますが、これが追加タイムラインやサーバー拡張で違う場合には書くタイムライン描画を編集する必要があります。

そのため、これの解決のために、オプションの一覧と、その排他性も共通ファイルの方で管理し、それを各タイムライン描画側で見るようにすると、今後の拡張性が上がるなと考えました。

また、遠い将来には、サードパーティクライアントでのタイムラインの表示も簡単にできるように、リファクタリングでまとめた情報を、 misskey サーバーから与えることによって可能にできたらいいなと考えていたりしますが、これ実装したPRはレビュー通るのかなぁ。

## まとめ

とりあえず昔よりはタイムラインの処理がまとまってると思うので、フォークの方々は好きなタイムラインを追加してみてください〜

[calender]: https://adventar.org/calendars/10208
[misskey-readme]: https://github.com/misskey-dev/misskey#readme
[misskey.niri.la]: https://misskey.niri.la/#pswp
[Vmimi Relay Timeline Fork]: https://github.com/anatawa12/misskey/tree/vmimi-relay-timeline/releases/?tab=readme-ov-file#vmimi-relay-timeline
[Vmimi Relay]: https://relay.virtualkemomimi.net/
[vrtl-servers-api]: https://relay.virtualkemomimi.net/api/servers
[dev-pr-12507]: https://github.com/misskey-dev/misskey/pull/12507
[dev-issue-13488]: https://github.com/misskey-dev/misskey/issues/13488
[dev-pr-13495]: https://github.com/misskey-dev/misskey/pull/13495

[^FTT]: Misskey Fanout Timeline Technology (https://misskey-hub.net/en/docs/for-admin/features/ftt/)
