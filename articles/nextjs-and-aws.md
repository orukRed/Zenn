---
title: "NextJsとAWSを連携させて画像投稿アプリを作る"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## はじめに

NextJsとAWSを連携させて画像投稿アプリを作る

## 使用技術

- NextJS 14.0.4
- material-ui 5


nextjsは以下でインスト。

```bash
npx create-next-app@latest #Nextjsのプロジェクト生成
npm run dev #localhost:3000にアクセスできることを確認する
```

MaterialUIのインストはここに従えばいいはず。
https://mui.com/material-ui/guides/nextjs/

公式で指示されてるものに加えて以下もインスト。
```bash
@emotion/react
```
