+++
date = "2015-02-18T00:18:38+09:00"
draft = true
title = "Go言語のCLIツールのpanicをラップしてクラッシュレポートをつくる"
+++

[mitchellh/panicwrap](https://github.com/mitchellh/panicwrap)

Go言語でpanicが発生したらどうしようもない．普通はちゃんとテストをしてそもそもpanicが発生しないようにする．しかし，クロスコンパイルして様々な環境に配布することを，もしくはユーザが作者が思ってもいない使いかたをすることを考慮すると，すべてを作者の想像力の範疇のテストでカバーし，panicをゼロにできるとは限らない．

panicの発生を放置すると，ユーザからすると何が起こったか分からない．Go言語を使ったことがあるユーザなら「あの表示」を見て，panicが起こったことがわかるかもしれない．

開発者からすると，そのpanicの詳しい発生状況を基に修正を行い，新たなテストケースを追加して二度とそのバグが発生しないようにしておきたい．


