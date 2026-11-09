---
title: "無料枠（Spark）の制限を回避！Base64とFirebase Storageを環境変数で自動切り替える画像保存"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","Vite","Bootstrap"]
published: true
published_at: 2026-11-15 00:00
---
# 【React/Firebase】無料枠（Spark）の制限を回避！Base64とFirebase Storageを環境変数で自動切り替える画像保存関数の実装

こんにちは！個人開発で Firebase を使ったWebアプリを構築する際、初期費用を抑えるために無料の **Sparkプラン** からスタートすることは多いでしょう。

しかし、無料プランのままでFirestoreに画像（手相や顔写真など）を保存しようとすると、「Firestoreの1ドキュメントあたり1MBの容量制限」という大きな壁にぶつかります。かといって最初から有料のBlazeプランにするのは少しハードルが高い……。

今回は、環境変数（フラグ）を切り替えるだけで、「開発・無料時はBase64で手軽にデータ化し、本格運用・有料時はFirebase Storageへ自動でアップロードする」ことができる、柔軟で汎用的な画像保存関数（`saveImage`）の設計・実装方法を解説します。

---

## 1. アプローチの全体像

今回実装するのは、引数や環境変数に応じて以下の2つの保存方式を自動で切り替える共通関数です。

1. **無料プラン（Sparkプラン / ローカル開発向け）:**
`FileReader API` を用いて画像ファイルを `Base64` 文字列に変換し、そのままデータベースに保存する。


2. **有料プラン（Blazeプラン / 本番運用向け）:**
Firebase Storage の指定パスへバイナリファイルをアップロードし、発行された公開URLを返す。



これをコード側で個別に書き換えるのではなく、**1つの共通関数の中に閉じ込め、環境変数で一括制御**できるようにします。

---

## 2. 環境変数の設定（.env）

まずはプロジェクトのルートにある `.env` ファイルに、ストレージの切り替えフラグを定義します。

```env
# 有料プラン(Blaze)の Storage を利用するかどうかのフラグ
# 初期開発や無料枠テスト時は false に設定
VITE_USE_PAID_STORAGE=false

```

---

## 3. saveImage 関数の実装（src/firebase.js）

Firebase の初期化を行っている `src/firebase.js` の中に、画像保存の分岐ロジックを持つ `saveImage` 関数を実装します。

```javascript
import { initializeApp } from "firebase/app";
import { getAuth } from "firebase/auth";
import { getFirestore } from "firebase/firestore";
import { getStorage, ref, uploadBytes, getDownloadURL } from "firebase/storage";

// Firebase設定・初期化の省略...
const app = initializeApp(firebaseConfig);

export const auth = getAuth(app);
export const db = getFirestore(app);
export const storage = getStorage(app);

/**
 * 画像保存処理の分岐用共通関数
 * @param {File} file - 保存する画像ファイル
 * @param {string} path - Storage 保存先パス
 * @param {boolean} isPaidPlan - 有料プラン(Blaze)かどうか
 * @returns {Promise<string>} 画像の URL または Base64 文字列
 */
export async function saveImage(file, path, isPaidPlan = false) {
  if (isPaidPlan) {
    // 【有料プラン: Firebase Storageへアップロード】
    const storageRef = ref(storage, path);
    await uploadBytes(storageRef, file);
    return await getDownloadURL(storageRef);
  } else {
    // 【無料プラン: Base64に変換して文字列として保持】
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => resolve(reader.result);
      reader.onerror = (error) => reject(error);
      reader.readAsDataURL(file);
    });
  }
}

```

---

## 4. コンポーネント側での呼び出しと活用

新規作成画面などのフロントエンド側では、環境変数から取得したフラグ（`isPaidPlan`）を引数に渡すだけで、意識することなく適切な保存処理を実行できます。

```javascript
import { auth, saveImage } from '../firebase'

// 環境変数から有料プラン(Blaze)のフラグを取得
const isPaidPlan = import.meta.env.VITE_USE_PAID_STORAGE === 'true'

// フォーム送信時などの処理
const handleImageUpload = async (file) => {
  const user = auth.currentUser;
  const path = `users/${user.uid}/fortunes/${Date.now()}`;

  // フラグに応じて、Base64かStorage URLが自動で返ってきます
  const imageUrl = await saveImage(file, path, isPaidPlan);
  
  console.log("保存完了:", imageUrl);
}

```

---

## 5. 運用時の注意点と選び方

* **無料プラン（Spark / `isPaidPlan = false`）:**
クレジット登録不要で、モックアップ作成や個人開発の初期テストを完全無料でスピーディーに行えます。ただし、高画質な写真を何枚も追加するとFirestoreの1MB上限にすぐ達するため、テスト時は軽量な画像を使用するのがコツです。


* **有料プラン（Blaze / `isPaidPlan = true`）:**
Firebase Storageへ保存されるため容量制限を気にせず高画質写真を扱えます。Blazeプランであっても毎月一定の無料枠（Storageは5GBまで等）が適用されるため、個人開発の範囲であれば実質ほぼコストをかけずに安全な運用が可能です。



---

## まとめ

今回は、環境変数のフラグ一つで画像保存のバックエンドを自動切り替える `saveImage` 関数の実装方法を解説しました。

「最初は無料で手軽に作り始め、スケールしたらインフラ側の設定を切り替える」という堅牢な設計にしておくことで、開発のフェーズに合わせた柔軟なアプリ運営が可能になります。ぜひ皆さんのプロジェクトでも取り入れてみてください！

---
### 💡 さらに詳しく学びたい方へ

この記事は、KindleおよびZennにて販売中の電子書籍『ゼロから作る！React×Firebase AI占いアプリ開発講座』の第2章から一部抜粋して再編集したものです。
> ここまでで紹介したFirebaseの柔軟な設計から、さらに一歩進んで、
> * Firebase Authentication（認証）と Cloud Firestore（データベース）の実装
> * Google Gemini API を駆使したマルチモーダルAI鑑定機能
> * 本番デプロイ手順と、APIキーを完全に隠すプロ仕様の構成（Cloud Functions）
全体の設計や実践的なコード一式は、各書籍ページよりご覧いただけます。

* **[Zenn本](https://zenn.dev/yasu-tech-35/books/ai-fortune-app-book) 『ゼロから作る！React×Firebase AI占いアプリ開発講座』**
  * Zenn上でブラウザからサクサク読み進めたい方はこちら
* **[Kindle版](https://www.amazon.co.jp/dp/B0HMD48B7G) 『ゼロから作る！React×Firebase AI占いアプリ開発講座』（Amazon）**
  * Kindleアプリや端末でオフライン読書・ハイライト管理したい方はこちら

> **書籍版の限定コンテンツ:**
> * アプリ全体の完全ソースコード（GitHubリポジトリ）
> * エラーハンドリングと環境構築の徹底解説

