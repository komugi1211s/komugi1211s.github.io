---
title: "Yarn Spinner Runtime C99 Port"
date: 2022-04-20T11:15:50+09:00
draft: true
---

[Github Link](https://github.com/komugi1211s/yarn-c99)

## About

ナラティブ・ダイアログ作成ツールである[Yarn Spinner](https://yarnspinner.dev/)のランタイムをC99にポートしたもの。

### Feature
 - シンプルなAPI。数個のファイルをIncludeするだけですぐに動作する
   - CMakeなどの複雑なソリューションが不要。
 - yarnc + csv ファイルを開き、ダイアログを提供する
 - Luaスタイルの関数登録
 - 適切なライフタイムを持つ文字列ハンドリング
 - ユーザースペースで使用可能なハッシュマップ同梱
 - ユーザースペースで使用可能なアリーナアロケーター同梱
