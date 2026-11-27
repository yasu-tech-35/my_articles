---
title: "【React × Firebase】未ログインユーザーを自動で弾く！安全な保護ルート（ProtectedRoute）の作り方"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","Firebase","ReactRouter","JavaScript","認証"]
published: true
published_at: 2026-11-30 00:00
---
# 【React × Firebase】未ログインユーザーを自動で弾く！安全な保護ルート（ProtectedRoute）の作り方

こんにちは！Webアプリケーションにログイン機能を実装したあと、次に必ず対応しなければならないのが**「ログインしていないユーザーからの不正アクセス制御」**です。

せっかくログイン画面を作っても、URL（例：`/` や `/settings/prompts` など）を直接ブラウザに入力されたときに中身が見えてしまったり、未認証のままデータベース（Firestore）へアクセスしてパーミッションエラーを起こしたりしては、安全なアプリとは言えません。

本記事では、Firebase Authentication の **`onAuthStateChanged`** と React Router の **`<Navigate>`** を組み合わせ、**未ログインユーザーを自動的にログイン画面へリダイレクトするカスタムコンポーネント（`ProtectedRoute`）の作り方**を解説します。

## 1. 今回のゴールと仕組み

今回実装する `ProtectedRoute` コンポーネントは、保護したいページ（ダッシュボードや設定画面など）をラップして以下のような流れで動作します。

1. **認証状態の確認中（`loading === true`）:** Firebaseがログイン状態を判定し終わるまで、ローディングスピナーを表示して画面のチラつきを防ぐ。
2. **未ログイン状態（`!user`）:** ログイン画面（`/login`）へ自動的に強制リダイレクトする。
3. **ログイン済み（`user` あり）:** 本来表示したい子コンポーネント（`children`）をそのまま描画する。

---

## 2. 保護ルート（`src/components/ProtectedRoute.jsx`）の実装

`src/components` フォルダ内に `ProtectedRoute.jsx` を新規作成し、以下のコードを記述します。

```
import React, { useEffect, useState } from 'react'
import { Navigate } from 'react-router-dom'
import { onAuthStateChanged } from 'firebase/auth'
import { auth } from '../firebase'

export default function ProtectedRoute({ children }) {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // Firebaseの認証状態の変化をリアルタイムに監視
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser)
      setLoading(false)
    })

    // コンポーネント破棄時に監視を解除（メモリリーク防止）
    return () => unsubscribe()
  }, [])

  // 1. 認証状態を確認中の場合はローディング表示
  if (loading) {
    return (
      <div className="d-flex justify-content-center align-items-center vh-100">
        <div className="spinner-border text-primary" role="status">
          <span className="visually-hidden">読み込み中...</span>
        </div>
      </div>
    )
  }

  // 2. 未ログインの場合はログイン画面へ自動転送
  if (!user) {
    return <Navigate replace to="/login"/>
  }

  // 3. ログイン済みの場合は保護されたページを表示
  return children
}

```

---

## 3. ルーティングへの組み込み（`src/App.jsx`）

作成した `ProtectedRoute` を使って、`App.jsx` 内の各ルート（URL）を保護します。公開ページである `/login` 以外のルートを `<ProtectedRoute>...</ProtectedRoute>` で囲むだけで適用完了です。

```
import React from 'react'
import { BrowserRouter, Routes, Route } from 'react-router-dom'
import Login from './pages/Login'
import Dashboard from './pages/Dashboard'
import DivinationNew from './pages/DivinationNew'
import DivinationDetail from './pages/DivinationDetail'
import PromptSettings from './pages/PromptSettings'
import Navbar from './components/Navbar'
import ProtectedRoute from './components/ProtectedRoute'

export default function App() {
  return (
    <BrowserRouter>
      <div style={{ paddingBottom: '70px' }}>
        <Routes>
          {/* 公開ルート（未ログインでもアクセス可能） */}
          <Route path="/login" element={<Login/>} />

          {/* 保護されたルート（未ログイン時は /login に自動リダイレクト） */}
          <Route
            path="/"
            element={
              <ProtectedRoute>
                <Dashboard/>
              </ProtectedRoute>
            }
          />
          <Route
            path="/divination/new"
            element={
              <ProtectedRoute>
                <DivinationNew/>
              </ProtectedRoute>
            }
          />
          <Route
            path="/divination/:id"
            element={
              <ProtectedRoute>
                <DivinationDetail/>
              </ProtectedRoute>
            }
          />
          <Route
            path="/settings/prompts"
            element={
              <ProtectedRoute>
                <PromptSettings/>
              </ProtectedRoute>
            }
          />
        </Routes>
      </div>
      <Navbar/>
    </BrowserRouter>
  )
}

```

---

## 4. 実装の重要ポイント解説

### ① なぜ `loading` 状態が必要なのか？

ブラウザをリロードした直後、Firebase Authentication がサーバーと通信して「このユーザーはログイン済みか？」を判定するまでにわずかなタイムラグが発生します。
もし `loading` 判定を入れずに `if (!user)` だけを書いてしまうと、**ログイン済みのユーザーであっても、リロードした瞬間に一瞬だけ `user` が `null` と判定され、毎回 `/login` 画面に弾き飛ばされてしまいます**。`loading` ステートで判定完了を待つことが、実用的な認証ガードを作る最大のコツです。

### ② `<Navigate to="/login" replace />` の `replace` 属性

リダイレクト時に `replace` を指定しておくことで、ブラウザの履歴（History）が上書きされます。これにより、未ログイン状態で弾かれたあとにブラウザの「戻る」ボタンを押しても、再び保護ページへ戻って無限ループになるのを防ぐことができます。

### ③ `useEffect` のクリーンアップ処理

`onAuthStateChanged` は監視リスナーを登録するため、`return () => unsubscribe()` を記述してコンポーネントのアンマウント時に監視を解除しています。

---

## 5. 動作確認の手順

実装できたら、以下の流れで正しく保護されているかテストしてみましょう。

1. ログアウトした状態（またはシークレットウィンドウ）で `http://localhost:5173/` に直接アクセスする。


2. URLが自動的に `http://localhost:5173/login` に切り替わることを確認する。


3. ログイン後、ダッシュボード画面でブラウザの再読み込み（F5）を行い、ログイン状態が維持されたまま画面が表示されることを確認する。

---

## まとめ

今回は、`onAuthStateChanged` と React Router を活用した安全な保護ルート（`ProtectedRoute`）の実装方法を解説しました。

このコンポーネントを1つ作っておくだけで、新しいページを増やしたときも `<ProtectedRoute>` で囲むだけでセキュリティを担保できるようになります。認証機能とセットで必ず導入しておきましょう！

---

> **🚀 さらに高度なアプリ開発・全コードを学びたい方へ**
> 
> 全コードや詳しい解説をまとめた技術書も公開していますので、ぜひあわせてチェックしてみてください！
> 📚 **『ゼロから作る！React×Firebase AI占いアプリ開発講座』**
> 👉 [Kindle / Zennのリンクをここに配置]

>この記事は、KindleおよびZennにて販売中の電子書籍『ゼロから作る！React×Firebase AI占いアプリ開発講座』の第4章から一部抜粋して再編集したものです。
> ここまでで紹介したFirebaseの柔軟な設計から、さらに一歩進んで、
> * Cloud Firestore によるユーザーごとのデータ分離保存とセキュリティルール設計
> * HTML5 File API を使ったスマホカメラ連携と、無料/有料プラン自動切替の画像保存ロジック
> * 公式 SDK `@google/genai` を使った画像＋プロフィールのマルチモーダルAI鑑定・対話型チャット
> * Firebase Hosting への本番デプロイ手順と、Cloud Functions によるAPIキー完全秘匿化構成

全体の設計や実践的なコード一式は、各書籍ページよりご覧いただけます。

* **[Zenn本](https://zenn.dev/yasu-tech-35/books/ai-fortune-app-book) 『ゼロから作る！React×Firebase AI占いアプリ開発講座』**
  * Zenn上でブラウザからサクサク読み進めたい方はこちら
* **[Kindle版](https://www.amazon.co.jp/dp/B0HMD48B7G) 『ゼロから作る！React×Firebase AI占いアプリ開発講座』（Amazon）**
  * Kindleアプリや端末でオフライン読書・ハイライト管理したい方はこちら

> **書籍版の限定コンテンツ:**
> * アプリ全体の完全ソースコード（GitHubリポジトリ）
> * エラーハンドリングと環境構築の徹底解説

### 🚀 React × Gemini API 実践開発シリーズ（好評発売中）

本記事の技術スタック（React / Gemini API）をベースに、さらに高度なAIエージェント開発や業務・分析アプリへステップアップできる実践書シリーズです。

「次に何を作ろうか？」と考えている方は、以下の実践開発ガイドもぜひチェックしてみてください。

1. **AI同士に議論・改善させて成果物を作るなら**
   📘 **『LangGraph×Geminiで構築する「AIマルチエージェント事業企画作成システム」開発ガイド』**
   「提案役」と「承認・レビュー役」のAIエージェントをLangGraphで連携させ、納得のいく事業企画書が自動で出来上がる仕組みをハンズオンで構築します。
   🔗 [Amazonで詳細を見る](https://www.amazon.co.jp/dp/B0HS2VCXD9)

2. **同じ技術スタック（React×Firebase×Gemini 2.5 Flash-Lite）で別アプリを作るなら**
   📘 **『React × Firebase × Gemini 2.5 Flash-Liteで作る！ AIエージェント駆動型・技術記事生成＆管理Webアプリ開発入門』**
   認証・Firestore保存・Gemini 2.5 Flash-Lite連携という本記事の王道構成を活かし、日々のアウトプットを劇的に効率化する「記事生成＆管理アプリ」を完成させます。
   🔗 [Amazonで詳細を見る](https://www.amazon.co.jp/dp/B0HPWN9PPW)

3. **数値データの可視化×AI分析に挑戦するなら**
   📘 **『React × Gemini APIで作る Webマーケティング&マクロ経済分析ダッシュボード』**
   ReactでのダッシュボードUI構築とGeminiによるインサイト抽出を組み合わせ、データ分析Webアプリの開発スキルを身につけます。
   🔗 [Amazonで詳細を見る](https://www.amazon.co.jp/dp/B0HKDLRB2P)