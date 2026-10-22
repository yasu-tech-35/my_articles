---
title: "【React + Bootstrap 5】スマホ操作性を劇的に高める「固定ボトムナビゲーションバー」のコンポーネント設計"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","Vite","Bootstrap"]
published: true
published_at: 2026-10-29 00:00
---
# 【React + Bootstrap 5】スマホ操作性を劇的に高める「固定ボトムナビゲーションバー」のコンポーネント設計

こんにちは！Webアプリケーションをスマートフォン向けに最適化する際、画面上部のヘッダーメニューだけでなく、片手操作でも指が届きやすい画面下部の「固定ボトムナビゲーションバー」を配置すると、アプリの操作性が格段に向上します。

今回は、`react-router-dom` と Bootstrap 5 を組み合わせて作る**レスポンシブなボトムナビゲーションの設計テクニック**と、初心者がハマりがちな「下部コンテンツの被り（見切れ）を防ぐパディング確保のコツ」を解説します。

---

## 1. 作成するボトムナビゲーションの仕様

今回作成するコンポーネントには、実用的な以下の機能を盛り込みます。

* **画面下部への固定表示:** CSSの `fixed-bottom` で下部に固定。


* **アクティブ状態の自動判定:** 現在開いているURL（パス）に応じて、該当するアイコンの色をハイライト表示。


* **特定画面での自動非表示:** ログイン画面（`/login`）など、ナビゲーションを出したくない画面では自動で非表示化。



---

## 2. Navbar コンポーネントの実装コード

`src/components/Navbar.jsx` を作成し、以下のコードを記述します。

```javascript
import React from 'react'
import { Link, useLocation } from 'react-router-dom'

export default function Navbar() {
  const location = useLocation()

  // 1. ログイン画面ではボトムナビゲーションを表示しない
  if (location.pathname === '/login') return null

  // 2. 現在のページに応じてアクティブ状態の判定とクラス切り替え
  const isActive = (path) => 
    location.pathname === path ? 'text-primary fw-bold' : 'text-secondary'

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

### コード解説のポイント

* **`useLocation()` の活用:**
`react-router-dom` が提供する `useLocation()` フックで現在のパスを取得し、`isActive()` 関数でアイコンの文字色を動的に変更しています。


* **`return null` による表示制御:**
認証前などの画面でコンポーネント自体を描画しない制御を行うことで、無駄なUI要素を排除できます。



---

## 3. 重要！「コンテンツ被り」を防ぐパディング確保のコツ

固定ナビゲーション（`fixed-bottom` や `position: fixed`）を導入した際に必ず直面するのが、「画面の一番下にあるボタンやテキストがナビゲーションバーの裏に隠れて押せなくなる」というレイアウトトラブルです。

### 解決策：親要素（`App.jsx`）で一括パディングを設定する

各ページコンポーネント側で個別に余白（`margin-bottom` や `padding-bottom`）を設定すると、管理が煩雑になり修正漏れの原因になります。

そのため、`App.jsx` などのルーティングを囲む親要素に対して、**ボトムナビゲーションの高さ（約60px〜70px）分だけ一括で `paddingBottom` を確保する**のが最もスマートで確実なアプローチです。

```javascript
// src/App.jsx の例
import React from 'react'
import { BrowserRouter, Routes, Route } from 'react-router-dom'
import Dashboard from './pages/Dashboard'
import DivinationNew from './pages/DivinationNew'
import Navbar from './components/Navbar'

export default function App() {
  return (
    <BrowserRouter>
      {/* ナビゲーションの高さ分の余白（paddingBottom）を一括確保！ */}
      <div style={{ paddingBottom: '70px' }}>
        <Routes>
          <Route path="/" element={<Dashboard />} />
          <Route path="/divination/new" element={<DivinationNew />} />
        </Routes>
      </div>
      
      {/* 下部固定のボトムナビゲーション */}
      <Navbar />
    </BrowserRouter>
  )
}

```

この設定を入れておくことで、ページスクロールの最下部でもコンテンツがボトムナビと被ることなく、最後の要素まできれいに閲覧・操作できるようになります。

---

## まとめ

今回は、スマホ向けWebアプリのUIをワンランク上げる**固定ボトムナビゲーションバーのコンポーネント設計**と、**コンテンツ被りを防ぐパディング処理のコツ**を解説しました。

ちょっとしたパディングの工夫ひとつで、実機での操作感やUXが大きく向上します。スマホ対応アプリを開発する際は、ぜひ取り入れてみてください！

---

### 💡 さらに詳しく学びたい方へ

この記事は、KindleおよびZennにて販売中の電子書籍『ゼロから作る！React×Firebase AI占いアプリ開発講座』の第2章から一部抜粋して再編集したものです。
> ここまでで紹介したスマホ向けUI設計から、さらに一歩進んで、
> * Firebase（Authentication / Firestore / Storage）による認証とデータ管理
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
