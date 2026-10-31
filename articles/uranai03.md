---
title: "【React】React Router（react-router-dom）で実現するスマホ向けマルチページUI構築術"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","Vite","Bootstrap"]
published: true
published_at: 2026-10-15 00:00
---
# 【React】React Router（react-router-dom）で実現するスマホ向けマルチページUI構築術：画面を再読み込みせずに高速切り替え

こんにちは！前回は Vite と React、そして Bootstrap 5 を使ってスマホ向けWebアプリの土台を作りました。

今回は第3回として、`react-router-dom` を導入し、ページを再読み込みすることなく瞬時に画面を切り替える「マルチページ構造（ルーティング）」の構築方法を解説します。さらに、スマートフォンでの操作性を劇的に向上させる「固定ボトムナビゲーションバー」の実装テクニックも合わせて紹介します。

---

## 1. 今回実現するルーティング構造

スマホ向けWebアプリでは、タブを切り替えるようにスムーズに画面が遷移してほしいものです。今回は以下の5つのページコンポーネントを想定したルーティングを組み立てます。

* ログイン画面 (`/login`)
* ダッシュボード・履歴一覧 (`/`)
* 新規占い画面 (`/divination/new`)
* 占い結果詳細・チャット (`/divination/:id`)※動的ルーティング
* プロンプト設定画面 (`/settings/prompts`)

## 2. react-router-dom のインストール

まずは、Reactでルーティングを行うための公式ライブラリをインストールします。
ターミナルで以下のコマンドを実行してください。

```bash
npm install react-router-dom

```

## 3. 各ページコンポーネントの骨組み準備

移行先となる各ページのコンポーネント（例：`Dashboard.jsx` や `DivinationDetail.jsx` など）を `src/pages/` フォルダ内に作成します。

特に、詳細画面のようにURLの一部が変化する箇所には、`react-router-dom` の `useParams` フックを使用して動的なIDを受け取れるようにします。

```javascript
// src/pages/DivinationDetail.jsx の例
import React from 'react'
import { useParams } from 'react-router-dom'

export default function DivinationDetail() {
  const { id } = useParams() // URLパラメータからIDを取得
  return (
    <div className="container py-3">
      <h2>鑑定結果・チャット分析</h2>
      <p>鑑定ID: <code>{id}</code></p>
    </div>
  )
}

```

## 4. スマホ向け「ボトムナビゲーション」の作成

スマホアプリでおなじみの、画面下部に固定されるナビゲーションバーを作成します。`src/components/Navbar.jsx` を作成しましょう。

```javascript
import React from 'react'
import { Link, useLocation } from 'react-router-dom'

export default function Navbar() {
  const location = useLocation()

  // ログイン画面ではボトムナビゲーションを表示しない
  if (location.pathname === '/login') return null

  // 現在のページに応じてアクティブ状態のスタイルを切り替え
  const isActive = (path) => location.pathname === path ? 'text-primary fw-bold' : 'text-secondary'

  return (
    <nav className="navbar fixed-bottom navbar-light bg-white border-top shadow-sm py-2">
      <div className="container d-flex justify-content-around text-center">
        <Link to="/" className={`text-decoration-none ${isActive('/')}`}>
          <div className="fs-5">📋</div>
          <small style={{ fontSize: '0.75rem' }}>履歴</small>
        </Link>
        <Link to="/divination/new" className={`text-decoration-none ${isActive('/divination/new')}`}>
          <div className="fs-5">✨</div>
          <small style={{ fontSize: '0.75rem' }}>新規占い</small>
        </Link>
        <Link to="/settings/prompts" className={`text-decoration-none ${isActive('/settings/prompts')}`}>
          <div className="fs-5">⚙️</div>
          <small style={{ fontSize: '0.75rem' }}>設定</small>
        </Link>
      </div>
    </nav>
  )
}

```

* **ポイント:** `useLocation()` フックを使うことで、現在どのURLにいるかをリアルタイムで判定し、アクティブなタブの色を変えるスタイリングを簡単に実装できます。また、ログイン画面など不要な場所では `return null` で非表示にしています。



## 5. App.jsx でルーティングを統合する

作成したページとナビゲーションを `BrowserRouter` で包み込み、URLに応じたルーティングを `App.jsx` に記述します。

```javascript
import React from 'react'
import { BrowserRouter, Routes, Route } from 'react-router-dom'
import Login from './pages/Login'
import Dashboard from './pages/Dashboard'
import DivinationNew from './pages/DivinationNew'
import DivinationDetail from './pages/DivinationDetail'
import PromptSettings from './pages/PromptSettings'
import Navbar from './components/Navbar'

export default function App() {
  return (
    <BrowserRouter>
      {/* ボトムナビゲーションの固定によるコンテンツ被りを防ぐため、下部に余白を確保 */}
      <div style={{ paddingBottom: '70px' }}>
        <Routes>
          <Route path="/login" element={<Login />} />
          <Route path="/" element={<Dashboard />} />
          <Route path="/divination/new" element={<DivinationNew />} />
          <Route path="/divination/:id" element={<DivinationDetail />} />
          <Route path="/settings/prompts" element={<PromptSettings />} />
        </Routes>
      </div>
      
      {/* 画面下部に常時固定されるボトムナビゲーション */}
      <Navbar />
    </BrowserRouter>
  )
}

```

---

## 6. 実装時のハマりどころ・注意点

1. **画面下のコンテンツがボトムナビに隠れてしまう問題:**
`fixed-bottom` でナビゲーションを固定すると、その下のコンテンツと重なって見えなくなる現象が起きます。上記コードのように、ラップしている親要素に `style={{ paddingBottom: '70px' }}` のような余白を必ず持たせるのがコツです。


2. **タグの代わりにを使う:**
従来の `<a href="...">` を使うとページ全体がリロードされてしまいますが、`react-router-dom` の `<Link to="...">` を使うことで、非同期（SPAならではの高速な画面切り替え）を実現できます。



---

## まとめ

今回は `react-router-dom` を使って、スマホアプリらしいスムーズなマルチページ構成とボトムナビゲーションを実装しました。画面のリロードなしでサクサク動くUIになると、アプリのクオリティが一気に上がりますね！

次回は、いよいよユーザー管理の要となる「Firebase Authentication（ログイン・会員登録機能）」の導入と、未ログインユーザーを自動ではじく保護ルーティングの構築に進みます！

---

### 💡 さらに詳しく学びたい方へ

この記事は、KindleおよびZennにて販売中の電子書籍『ゼロから作る！React×Firebase AI占いアプリ開発講座』の第2章から一部抜粋して再編集したものです。
> ここまでで紹介したルーティング設計から、さらに一歩進んで、
> * Firebase（Authentication / Firestore / Storage）による安全な認証とデータ管理
> * Google Gemini API を駆使したマルチモーダルAI鑑定・対話チャット機能
> * 本番デプロイとAPIキーの完全秘匿化（Cloud Functions）
全体の設計や実践的なコード一式は、各書籍ページよりご覧いただけます。

* **[Zenn本](https://zenn.dev/yasu-tech-35/books/ai-fortune-app-book) 『ゼロから作る！React×Firebase AI占いアプリ開発講座』**
  * Zenn上でブラウザからサクサク読み進めたい方はこちら
* **[Kindle版](https://www.amazon.co.jp/dp/B0HMD48B7G) 『ゼロから作る！React×Firebase AI占いアプリ開発講座』（Amazon）**
  * Kindleアプリや端末でオフライン読書・ハイライト管理したい方はこちら

> **書籍版の限定コンテンツ:**
> * アプリ全体の完全ソースコード（GitHubリポジトリ）
> * エラーハンドリングと環境構築の徹底解説