---
title: "【Kotlin】Androidアプリでボタン連打を抑止する方法（DataBinding編）"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Android","Kotlin","DataBinding"]
published: false
---
この記事は[Kotlin Advent Calendar 2020](https://qiita.com/advent-calendar/2020/kotlin)の5日目の記事です。
Androidアプリ開発初心者ですが、空きがあったので勢いだけで参加してみました。(1日ぶり2回目)

## はじめに
Androidに限らず色々なアプリを開発していて、ついつい忘れがちなボタン連打対策。。。
処理の多重実行によってバグが発生することも多々あるため対処が必要です。

いざ実装しようと調べて見ると、通常の連打対策の記事はよくありますが、
DataBindingを利用しているプロジェクトで、簡単に導入できる方法が見つからなかったため、備忘録として記事にしようと思います。

## DataBindingを使った場合の通常のボタン押下処理
`android:onClick`を利用してボタン押下時処理と紐付けを行います。

```xml:MainLayout.xml
<?xml version="1.0" encoding="utf-8"?>
<layout
  xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:app="http://schemas.android.com/apk/res-auto"
  xmlns:tools="http://schemas.android.com/tools">
  <data>
    <variable name="viewModel" type="com.sample.viewmodel.MainViewModel" />
  </data>
  <LinearLayout>
    <Button
      android:onClick="@{() -> viewModel.onClickButton()}"
    />
  </LinearLayout>
</layout>
```
※ 実際は幅、高さなどの属性も指定しないとエラーになりますが、今回の説明に不要な部分は省略してあります。

## 連打対策済みボタン押下時処理
Extensionを利用し全てのビューの基底クラスに
OnClickListenerのセッターのラッパー(連打防止が処理入り)を定義し、
カスタム属性と紐付けます。

```Kotlin:ViewExtension.kt
package com.sample.extension

import android.view.View
import androidx.databinding.BindingAdapter
/**
 * ==============================================
 *  View Extensions
 * ==============================================
 */
/**
 * setOnSingleClickListener
 */
@BindingAdapter("android:onSingleClick")
fun View.setOnSingleClickListener(listener: View.OnClickListener) {
  // 押下後一時的に押下処理を無効化する時間(ms)
  val delayMillis = 500L
  // 前回押下時間(タイムスタンプ, ms)
  var pushedAt = 0L

  this.setOnClickListener {
    if (System.currentTimeMillis() - pushedAt < delayMillis) {
      // 前回押下から規定時間分経過していない場合は、処理終了
      return@setOnClickListener
    }
    // 押下時間を更新
    pushedAt = System.currentTimeMillis()
    // 押下時処理を実行
    listener.onClick(this)
    return@setOnClickListener
  }
}
```

レイアウトでの利用方法は`android:onClick`の代わりに利用するだけです。

```xml:MainLayout.xml
<?xml version="1.0" encoding="utf-8"?>
<layout
  xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:app="http://schemas.android.com/apk/res-auto"
  xmlns:tools="http://schemas.android.com/tools">
  <data>
    <variable name="viewModel" type="com.sample.viewmodel.MainViewModel" />
  </data>
  <LinearLayout>
    <Button
      android:onSingleClick="@{() -> viewModel.onClickButton()}"
    />
  </LinearLayout>
</layout>
```

## さいごに
次からは初期段階から導入するように注意します....
~~私の場合、気づいたときには、`android:onClick`が既に100箇所以上定義されていたので、少しの書き換えで済んで良かった。。。~~

## 参考記事
- [ボタンの連打を防止する処理を共通化する - コジオニルク](https://kojion.com/posts/1057)
