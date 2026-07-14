---
title: "第2部 Componentの基礎"
---

第2部では、Angularアプリケーションの中心にある「Component」を学びます。前の第1部では、Angularの全体像と、プロジェクトが起動するまでの流れを見てきました。ここからは、その画面を実際に組み立てる主役であるComponentに焦点を当てます。

Angularの画面は、Componentという部品を組み合わせて作られます。ボタンひとつ、フォームひとつ、ページ全体まで、すべてがComponentです。この部では、Componentが何でできているのか、どう作るのか、そしてどう分割・設計すればよいのかまでを、順を追って解説します。あわせて、Componentを支える仕組みであるStandaloneやコンテンツ投影、スタイルのカプセル化も扱います。

## この部で学ぶこと

- **第5章 Angularで使うTypeScript**: Componentを書くうえで前提となるTypeScriptの要点を整理します。
- **第6章 Componentとは何か**: Componentの構成要素と、その作り方・つなぎ方を学びます。
- **第7章 NgModuleからStandalone Componentへ**: 従来のNgModuleと、現在の標準であるStandalone Componentを比較します。
- **第8章 コンテンツ投影とクエリ**: `ng-content`による内容の差し込みと、`viewChild()`・`contentChild()`による子要素の参照を扱います。
- **第9章 スタイリングとView Encapsulation**: Componentのスタイルがどう分離されるのか、その仕組みと使い分けを学びます。
- **第10章 Componentの分割と責務**: Componentをどう分割し、責務をどう配分するかという設計の指針を扱います。

## 前提となる知識

この部を読むにあたって、次を前提とします。

- 第1部の内容（Angular CLIでプロジェクトを作成し、開発サーバーで表示できること）
- HTML・CSSの基本的な書き方

TypeScriptの前提知識は不要です。必要な範囲は第5章で解説します。

## 対象レベル

**基礎**。Angularでの画面づくりの土台を身につける部です。データバインディングやDIなど、より踏み込んだ仕組みは第3部以降で扱うため、まずはComponentそのものの理解に集中します。

## まとめ

- 第2部では、Angularの画面を組み立てる中心的な部品であるComponentを学びます
- Componentの構成・作り方から、Standalone・コンテンツ投影・スタイル・設計までを順に扱います
- 前提は第1部の内容とHTML・CSSの基礎で、TypeScriptは第5章で補います

次の第5章では、Componentを書く準備として、Angularで使うTypeScriptの要点を整理します。
