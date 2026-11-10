# 画像・参考資料

- Bob キャラクター：IBM公式 https://bob.ibm.com/ （https://bob.ibm.com/assets/bob-standing-BECrMjXJ.webp）。IBMのブランド画像。
- 筋トレアプリのアイコン：公開サイトの favicon / apple-touch-icon https://muscle-training-mcp.tsukutakun.workers.dev/icon.png?v=2 。余白を除き、背景を透過して使用。
- アイコン：Lucide https://lucide.dev/ 、ISC License。assets/lucide-LICENSE を同梱。
- https://github.com/ikuoikuo/muscle-training-mcp
- https://qiita.com/ikuoikuo/items/aeae10406c4730ab67cf
- https://qiita.com/ikuoikuo/items/1352d6652bfd94d36933
- Issue #141 とユーザー提供の IBM Bob MCP OAuth 記事案

図の「筋トレアプリ」は muscle-training-mcp。MCPの双方向矢印はツール呼び出しと応答。MCP接続線のOAuth表記は、そのアクセスを認可する仕組みを表す。

## 今回の記事で使用する画像（ユーザー提供、2026-11-10取得）

記事は以下のローカル画像を参照。認証画面は個人情報を不透明に塗りつぶした派生画像を使用している。元の公開URL自体は変更していない。

| 保存ファイル | 元画像ID（Qiita画像ストア） |
| --- | --- |
| relationship.png | a2ed19eb-e221-4351-990b-64a158e60980 |
| record-example.png | 318d1ba3-9fec-48dc-a112-fb8c79f62d86 |
| open-settings.png | f2b6969a-c603-43b7-a346-6a4af872b44d |
| mcp-tab.png | 78dd4e06-5c08-4ba9-bd81-5d2ee471a8ac |
| authenticate.png | 436325c5-1536-4ef7-a50b-f41b23d88796 |
| google-sign-in.png | 44b9806e-1fb5-468c-a1d3-b81cb00e8032 |
| authorize-access.png | 3f5b4f8b-b31e-4493-9720-24a8e5c344c7 |

元画像URLの形式：`https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/{ID}.png`

## 追加レポート画像（ユーザー提供）

| 保存ファイル | 元画像ID（Qiita画像ストア） |
| --- | --- |
| report-frequency.png | 441ca51b-101a-4694-be0e-102863297b03 |
| report-weights.png | 6518ca06-dcfa-4db7-9607-82bf7b84280b |
| report-feedback.png | ca92828e-bd4c-42dc-af58-7f3f4388f0c1 |

3枚とも目視確認し、メールアドレスなどの個人識別情報は見当たらなかったため、マスキングは追加していない。

## 手順画像の注釈・表示調整

設定手順の5枚は、元画像を残して `*-annotated.png` を作成。操作対象を赤枠（白い縁取り付き）で強調し、設定画面は操作に必要な範囲へ切り抜いた。個人情報のマスキングは維持。画像ファイル自体の幅を設定アイコン680px、MCPタブ・認証・Googleログイン760px、縦長のアクセス許可画面420pxに縮小。HTMLタグが文字として表示されるプレビューに対応するため、記事では通常のMarkdown画像記法を使用する。
