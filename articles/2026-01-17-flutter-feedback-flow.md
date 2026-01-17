---
title: "【Flutter】ストアのレビューを増やすためにIn-App ReviewとKoeLoopを組み合わせた個人的最適解"
emoji: "🔄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "dart", "個人開発", "ui", "ux"]
published: false
---

個人開発している育児記録アプリ「**[milu](https://babymom-diary.web.app//)**」で、ユーザレビューとフィードバック収集のフローを改善しました。
きっかけは、以前Xで見かけたこちらのポストです。

https://x.com/Nyantaro_Kagura/status/1753329768801075408?s=20

このポストで紹介されていた**「満足しているユーザにはストアレビューを促し、不満があるユーザには意見箱（フィードバックフォーム）へ案内する」**というアプローチが非常に合理的だと感じ、自分のアプリにも取り入れてみました。

本記事では、Flutterアプリでの実装例を紹介します。

## 実装の概要

全体のフローは以下のようになります。

1.  **タイミングを見計らってダイアログを表示**
    *   ある程度アプリを使ってくれているユーザ（例: 起動回数や記録回数が一定以上）に対して、「アプリに満足していますか？」と尋ねるダイアログを表示します。
2.  **分岐**
    *   **「満足」**を選択した場合:
        *   `in_app_review` パッケージを使って、アプリ内でストアレビューをリクエストします。
    *   **「不満」**を選択した場合:
        *   「改善のためにご意見をお聞かせください」と促し、フィードバック収集サービス（今回は **KoeLoop** を使用）へ誘導します。

## 1. 満足度確認ダイアログの実装

まずはユーザの感情を確認するシンプルなダイアログを作成します。
iOS/Androidそれぞれのプラットフォームに合わせたデザイン（`CupertinoAlertDialog` / `SimpleDialog`）で出し分けると自然です。

```dart
// 簡略化した実装イメージ
Future<void> showSatisfactionDialog(BuildContext context) async {
  final result = await showDialog<bool>(
    context: context,
    builder: (context) => AlertDialog(
      title: const Text('アプリに満足いただけていますか？'),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, true), // 満足
          child: const Text('満足'),
        ),
        TextButton(
          onPressed: () => Navigator.pop(context, false), // 不満
          child: const Text('不満'),
        ),
      ],
    ),
  );

  if (result == true) {
    // 満足 -> ストアレビューへ
    _requestStoreReview();
  } else if (result == false) {
    // 不満 -> フィードバックへ
    _requestFeedback(context);
  }
}
```

:::message
実際には、誤操作を防ぐためのUI工夫や、クロスプラットフォーム対応（`dart:io` の `Platform.isIOS` 分岐など）を入れるとより良いでしょう。
:::

## 2. 「満足」の場合: In-App Review

満足しているユーザには、その場で星をつけてもらえるよう `in_app_review` パッケージを使用します。

https://pub.dev/packages/in_app_review

```dart
import 'package:in_app_review/in_app_review.dart';

Future<void> _requestStoreReview() async {
  final InAppReview inAppReview = InAppReview.instance;

  if (await inAppReview.isAvailable()) {
    // アプリ内でレビューダイアログを表示
    await inAppReview.requestReview();
  } else {
    // フォールバック: ストアのページを開く
    // await inAppReview.openStoreListing(appStoreId: '...', microsoftStoreId: '...');
  }
}
```

「満足」と答えた直後であれば、高い評価をいただける可能性が高まります。

## 3. 「不満」の場合: KoeLoopへ誘導

不満を感じているユーザには、ストアで低評価を書かれる前に、直接開発者に意見を届けるルートを用意します。
今回は、ユーザの声を集めるプラットフォーム **[KoeLoop](https://koeloop.dev/)** を利用しました。

アプリ内ブラウザ（WebView）で開くことで、アプリから離脱せずに意見を投稿してもらえます。

https://pub.dev/packages/url_launcher

```dart
import 'package:url_launcher/url_launcher.dart';

Future<void> _requestFeedback(BuildContext context) async {
  // まずはワンクッション置いて、意見を送るか確認
  final sendFeedback = await showDialog<bool>(
    context: context,
    builder: (context) => AlertDialog(
      title: const Text('改善のためにご意見をお聞かせください'),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, true),
          child: const Text('意見を送る'),
        ),
        TextButton(
          onPressed: () => Navigator.pop(context, false),
          child: const Text('今はしない'),
        ),
      ],
    ),
  );

  if (sendFeedback == true) {
    // KoeLoopのURLを開く（InAppWebViewモード推奨）
    final uri = Uri.parse('https://koeloop.dev/embed/YOUR_ID?theme=light...');
    await launchUrl(uri, mode: LaunchMode.inAppWebView);
  }
}
```

KoeLoopは埋め込みやすく、シンプルなUIで意見を集められるため重宝しています。

## まとめ

このフローを導入することで、以下の効果を期待しています。

*   **ストア評価の向上**: 満足しているユーザを確実にレビューへ誘導。
*   **具体的フィードバックの獲得**: 不満を持つユーザのガス抜きと、具体的な改善点の収集。

ユーザ体験を損なわずに、開発者にとっても有益な情報を集める良い仕組みだと感じています。
ぜひ試してみてください。

## 最後に

今回紹介した「milu」はこちらからダウンロードできます。
育児記録アプリですが、デザインや使い勝手にはかなりこだわって作っています。
Flutterエンジニアの方も、ぜひ触ってみて感想をいただけると嬉しいです！

<div style="display:flex; gap:10px;">
  <a href="https://apps.apple.com/jp/app/%E8%82%B2%E5%85%90%E8%A8%98%E9%8C%B2-%E4%BA%88%E9%98%B2%E6%8E%A5%E7%A8%AE%E7%AE%A1%E7%90%86-milu/id6754955821?l=en-US">
    <img src="/images/app_store/Download_on_the_App_Store_Badge_JP_RGB_blk_100317.svg" width="135" alt="Download on the App Store">
  </a>
  <a href="https://play.google.com/store/apps/details?id=com.aphlo.babymomdiary">
    <img src="/images/app_store/GetItOnGooglePlay_Badge_Web_color_Japanese.svg" width="150" alt="Get it on Google Play">
  </a>
</div>

---

また、X（旧Twitter）でも個人開発に関する情報を発信しているのでよければフォローしてください！励みになります。

https://x.com/aphlooo
