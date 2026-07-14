---
title: "第9部 FormsとHTTP通信"
---

第9部では、アプリケーションと外界をつなぐ2つの要、FormsとHTTP通信を学びます。Formsは、利用者からの入力を受け取る仕組みです。HTTP通信は、サーバーとデータをやり取りする仕組みです。この2つがあって初めて、アプリケーションは、利用者の操作を受け止め、データを保存し、必要な情報を取り寄せられます。

Formsもデータ通信も、Angularの歴史の中で進化を続けてきた領域です。Formsには、古くからあるTemplate-driven方式とReactive方式に加え、Angular 22でSignal Formsという新しい選択肢が加わりました。データ通信も、`HttpClient`に加えて、Signalベースの`httpResource()`が登場しています。この部では、それぞれの新旧を比較しながら、モダンな書き方を身につけます。

## この部で学ぶこと

- **第43章 Template-driven FormsとReactive Forms**: 2つの伝統的なフォーム方式を学びます。
- **第44章 Typed FormsとSignal Forms**: 型付きフォームと、v22の新しいSignal Formsを扱います。
- **第45章 HttpClientとAPI通信**: サーバーとの通信の基本を学びます。
- **第46章 HttpClientから`resource()`・`httpResource()`へ**: Signalベースのデータ取得を、新旧比較で扱います。
- **第47章 Interceptor・認証・エラー・ローディング設計**: 通信にまつわる横断的な設計を学びます。

## 前提となる知識

この部を読むにあたって、次を前提とします。

- 第8部の内容（RxJS、とくにObservableと`async`パイプ）
- Signal（第6部）と、Service／DI（第5部）

## 対象レベル

**中級**。フォームと通信は、実務のアプリケーションで必ず使う領域です。新旧の方式が併存しているため、どれをいつ使うのかの判断も含めて解説します。

## まとめ

- 第9部では、入力を受け取るFormsと、サーバーと通信するHTTPを学びます
- Formsは、Template-driven・Reactive・Signal Formsの3方式を比較します
- 通信は、`HttpClient`と、Signalベースの`httpResource()`を扱います

次の第43章では、まず伝統的な2つのフォーム方式、Template-drivenとReactiveから始めます。
