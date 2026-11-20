---
title: "React + Firebase】初心者でもわかる！Firebase Authenticationで「メール/パスワード認証」と「Googleログイン」を実装する方法"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","Vite","Bootstrap","Firebase"]
published: true
published_at: 2026-11-22 00:00
---
# 【React + Firebase】初心者でもわかる！Firebase Authenticationで「メール/パスワード認証」と「Googleログイン」を実装する方法

こんにちは！Webアプリケーションを作る際、「ログイン・会員登録機能」を自前でゼロから作ろうとすると、パスワードの暗号化やセッション管理、セキュリティ対策などでとても大変です。

そこで活躍するのが **Firebase Authentication** です。Firebaseを使えば、安全で信頼性の高い認証機能を驚くほど少ないコードで実装できます。

今回は、**Firebaseコンソールでの認証プロバイダの有効化**から、**Reactでのログイン・新規登録画面（`Login.jsx`）の実装**、そして**初心者がつまずきやすいエラーの対処法**まで、ステップ・バイ・ステップで詳しく解説します。

---

## 1. 今回のゴールと事前準備

この記事を読み終えると、以下のことができるようになります。

* Firebaseコンソールで「メール/パスワード認証」と「Googleログイン」を有効にできる


* メールアドレス・パスワードでの新規登録とログインができる画面を作れる


* ワンクリックでサインインできる「Googleログイン」を組み込める



※ 事前に `npm install firebase` でFirebase SDKをインストールし、`src/firebase.js` で `auth` インスタンスをエクスポートしておく必要があります。

---

## 2. Firebaseコンソールでの認証プロバイダ有効化

まずは、Firebaseの管理画面側で「どのログイン方法を使うか」を許可します。

### ① Authentication 画面へ移動する

1. [Firebase コンソール](https://console.firebase.google.com/) にアクセスし、対象のプロジェクトを選択します。


2. 左側のメニューから **「構築」＞「Authentication」** をクリックします。


3. 案内画面が表示されたら **「始める」** ボタンを押します。



### ② メール/パスワード認証を有効にする

1. **「ログイン方法」** タブ（または「Sign-in method」）から **「メール/パスワード」** をクリックします。


2. 一番上の **「有効にする」** スイッチをオン（有効）にします。


3. 右下の **「保存」** をクリックします。



### ③ Googleログインを有効にする

1. 同じ画面の **「新しいプロバイダを追加」**（または「ログイン方法」一覧）から **「Google」** を選択します。


2. **「有効にする」** スイッチをオンにします。


3. 「プロジェクトの公開サポートメール」に自分のGoogleメールアドレスを選択し、**「保存」** をクリックします。



これでFirebase側の準備は完了です！

---

## 3. ログイン・会員登録画面（`src/pages/Login.jsx`）の実装

続いて、React側でログインと新規登録を切り替えて操作できる画面を作成します。

初心者の方でもそのまま動かせるよう、`src/pages/Login.jsx` の完全なコードを掲載します。

```jsx
import React, { useState } from 'react'
import { useNavigate } from 'react-router-dom'
import {
  signInWithEmailAndPassword,
  createUserWithEmailAndPassword,
  signInWithPopup,
  GoogleAuthProvider
} from 'firebase/auth'
import { auth } from '../firebase'

export default function Login() {
  const navigate = useNavigate()
  const [isRegister, setIsRegister] = useState(false) // false: ログイン / true: 新規登録
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')
  const [loading, setLoading] = useState(false)

  // メールアドレス・パスワードによる認証処理（新規登録 or ログイン）
  const handleAuth = async (e) => {
    e.preventDefault() // フォーム送信によるページリロードを防止
    setError('')
    setLoading(true)

    try {
      if (isRegister) {
        // 新規アカウント作成
        await createUserWithEmailAndPassword(auth, email, password)
      } else {
        // 既存ユーザーのログイン
        await signInWithEmailAndPassword(auth, email, password)
      }
      navigate('/') // 成功したらトップ（ダッシュボード）へ遷移
    } catch (err) {
      console.error(err)
      // Firebaseのエラーコードに応じて、わかりやすい日本語メッセージに変換
      if (err.code === 'auth/email-already-in-use') {
        setError('このメールアドレスは既に登録されています。')
      } else if (
        err.code === 'auth/wrong-password' ||
        err.code === 'auth/user-not-found' ||
        err.code === 'auth/invalid-credential'
      ) {
        setError('メールアドレスまたはパスワードが正しくありません。')
      } else if (err.code === 'auth/weak-password') {
        setError('パスワードは6文字以上で設定してください。')
      } else {
        setError('認証に失敗しました: ' + err.message)
      }
    } finally {
      setLoading(false)
    }
  }

  // Googleポップアップログイン処理
  const handleGoogleLogin = async () => {
    setError('')
    try {
      const provider = new GoogleAuthProvider()
      await signInWithPopup(auth, provider)
      navigate('/') // 成功したらトップへ遷移
    } catch (err) {
      console.error(err)
      setError('Googleログインに失敗しました: ' + err.message)
    }
  }

  return (
    <div className="container py-5" style={{ maxWidth: 420 }}>
      <div className="card p-4 shadow-sm border-0 rounded-3">
        <h2 className="text-center h4 mb-3 fw-bold text-primary">
          Webアプリ
        </h2>
        <h3 className="text-center h6 mb-4 text-muted">
          {isRegister ? '新規アカウント登録' : 'ログイン'}
        </h3>

        {/* エラーメッセージ表示エリア */}
        {error && (
          <div className="alert alert-danger py-2 small" role="alert">
            {error}
          </div>
        )}

        <form onSubmit={handleAuth}>
          <div className="mb-3">
            <label className="form-label small fw-bold">メールアドレス</label>
            <input
              type="email"
              className="form-control"
              required
              placeholder="example@mail.com"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
            />
          </div>

          <div className="mb-3">
            <label className="form-label small fw-bold">パスワード</label>
            <input
              type="password"
              className="form-control"
              required
              placeholder="6文字以上のパスワード"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
            />
          </div>

          <button
            type="submit"
            className="btn btn-primary w-100 py-2 mb-3 fw-bold"
            disabled={loading}
          >
            {loading ? '処理中...' : isRegister ? '登録する' : 'ログイン'}
          </button>
        </form>

        <div className="d-flex align-items-center my-3">
          <hr className="flex-grow-1" />
          <span className="px-2 text-muted small">または</span>
          <hr className="flex-grow-1" />
        </div>

        {/* Googleログインボタン */}
        <button
          type="button"
          className="btn btn-outline-danger w-100 py-2 mb-3 d-flex align-items-center justify-content-center gap-2 fw-bold"
          onClick={handleGoogleLogin}
        >
          <span>G</span>
          <span>Googleでログイン</span>
        </button>

        {/* ログインモードと新規登録モードの切り替えリンク */}
        <div className="text-center mt-2">
          <button
            type="button"
            className="btn btn-link btn-sm text-decoration-none"
            onClick={() => {
              setIsRegister(!isRegister)
              setError('')
            }}
          >
            {isRegister
              ? 'すでにアカウントをお持ちの方はこちら（ログイン）'
              : 'アカウントをお持ちでない方はこちら（新規登録）'}
          </button>
        </div>
      </div>
    </div>
  )
}

```

---

## 4. 初心者向け・コードのポイント解説

1. **`isRegister` ステートで1画面に集約:**
ログイン画面と新規登録画面を別々に作らなくても、`isRegister`（`true` / `false`）を切り替えるだけで、見出しやボタンの文言、呼び出すFirebase関数（`createUserWithEmailAndPassword` か `signInWithEmailAndPassword` か）をスマートに切り替えています。


2. **`GoogleAuthProvider` と `signInWithPopup`:**
Googleログインでは、`new GoogleAuthProvider()` を作り、`signInWithPopup(auth, provider)` を呼ぶだけで、Googleアカウント選択のポップアップウィンドウが開き、認証が完了します。


3. **エラーコード（`err.code`）の日本語化:**
Firebaseから返るエラーは英語のコード（`auth/weak-password` や `auth/email-already-in-use` など）になっているため、`if / else if` で分岐してユーザーに親切な日本語メッセージへ変換しています。



---

## 5. よくあるつまずきポイント・トラブルシューティング

* **`auth/operation-not-allowed` エラーが出る場合:**
Firebaseコンソールの「Authentication」＞「ログイン方法」で、「メール/パスワード」や「Google」が**有効（オン）になっていない**可能性があります。設定を見直して「保存」を押したか確認してください。


* **Googleログインのポップアップが開かない・すぐ閉じる場合:**
ブラウザのポップアップブロック機能や、広告ブロッカーなどの拡張機能が影響していることがあります。一時的にオフにして動作を確認してみてください。


* **パスワードが6文字未満でエラーになる場合:**
Firebase Authenticationの仕様上、パスワードは**最低6文字以上**が必要です。短い文字列を入力すると `auth/weak-password` になります。



---

## まとめ

今回は、Firebase Authenticationを使って「メール/パスワード認証」と「Googleログイン」を実装する一連の流れを解説しました。

認証機能が動くようになると、一気に「自分のWebアプリ」という実感が湧いてきますね！次は、ログインしていないユーザーを自動でログイン画面へ誘導する「保護ルート（`ProtectedRoute`）」と組み合わせて、より安全なアプリへと仕上げていきましょう。

---

> **🚀 さらに高度なアプリ開発・全コードを学びたい方へ**
> 
> 全コードや詳しい解説をまとめた技術書も公開していますので、ぜひあわせてチェックしてみてください！
> 📚 **『ゼロから作る！React×Firebase AI占いアプリ開発講座』**
> 👉 [Kindle / Zennのリンクをここに配置]

>この記事は、KindleおよびZennにて販売中の電子書籍『ゼロから作る！React×Firebase AI占いアプリ開発講座』の第2章から一部抜粋して再編集したものです。
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
