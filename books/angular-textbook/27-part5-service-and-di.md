---
title: "第5部 ServiceとDependency Injection"
---

第5部では、Componentから処理を切り出して受け持つServiceと、それをComponentへ届ける仕組みであるDependency Injection（依存性の注入、以下DI）を学びます。ここからは、本書の対象レベルが「中級」に移ります。Componentの書き方を一通り身につけた読者が、アプリケーションの構造を理解するための部です。

Componentは、画面の見た目と操作を担う部品でした。しかし、データの取得や業務ロジックまでComponentに詰め込むと、第10章で触れた「神Component」に近づきます。そうした関心事を切り出す受け皿がServiceです。そして、作ったServiceをComponentが受け取る手段が、AngularのDIです。DIはAngularの設計の根幹をなす仕組みで、これを理解すると、フレームワークの見え方が一段深まります。

## この部で学ぶこと

- **第22章 ServiceとComponentの責務**: 何をComponentに残し、何をServiceへ切り出すのかを整理します。
- **第23章 Dependency Injectionとは何か**: 依存を「注入してもらう」という考え方と、その利点を学びます。
- **第24章 Constructor Injectionから`inject()`へ**: 依存を受け取る書き方を、新旧比較で学びます。
- **第25章 ProviderとInjectorの階層**: Serviceがどこで作られ、どう共有されるのかという仕組みを扱います。

## 前提となる知識

この部を読むにあたって、次を前提とします。

- 第2部〜第4部の内容（Componentの構成・データフロー・`input()`／`output()`）
- クラスとインターフェースの基本的な理解（第5章）

## 対象レベル

**中級**。Componentを書けることを前提に、アプリケーションを支える構造を学ぶ部です。DIは最初は抽象的に感じられますが、具体例を通して、なぜこの仕組みが便利なのかを段階的に理解していきます。

## まとめ

- 第5部では、処理を受け持つServiceと、それを届けるDIを学びます
- Componentから関心事を切り出す先がServiceで、DIがそれをComponentへつなぎます
- DIはAngularの根幹をなす仕組みで、理解するとフレームワークの見え方が深まります

次の第22章では、まずServiceとは何か、Componentとどう責務を分けるのかから始めます。
