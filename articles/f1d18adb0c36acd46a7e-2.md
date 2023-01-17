---
title: "CSSで表示アニメーションを指定する方法"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CSS"]
published: true
publication_name: "no4_dev"
---

モーダル表示時などに普通にアニメーションを設定しても反映されなかったので、その対応策です。
実際の動作は、[ミニマルなモーダルライブラリを npm で公開しました](https://zenn.dev/ohnaka0410/articles/1b2f04fa529ca373740d) のデモで使用していますのでご確認ください。

## 表示アニメーションが効かないケース

```html:html
<input type="checkbox" class="trigger">
<div class="modal">
  <div class="modal__container"></div>
</div>
```

```css:css
.modal {
  display: none;
  /* opacity をアニメーションさせたい */
  opacity: 0;
  transition: opacity .3s ease;
}

.trigger:checked + .modal {
  /* チェックボックスがONになると表示される */
  display: block;
  /* アニメーションが効かない */
  opacity: 1;
}
```

## 表示アニメーションが効くケース

```html:html
同上
```

```css:css
.modal {
  display: none;
}

.modal .modal__container {
  /* opacity をアニメーションさせたい */
  opacity: 0;
  transition: opacity .3s ease;
}

.trigger:checked + .modal {
  /* チェックボックスがONになると表示される */
  display: block;
}

.trigger:checked + .modal .modal__container {
  /* displayプロパティを変更した要素の内側の要素であれば、 */
  /* (displayプロパティを変更した要素以外の要素であれば、) */
  /* アニメーションが効く */
  opacity: 1;
}

```

## 注意

- 非表示は、アニメーションが実行される前に、display: none;になってしまうためアニメーションが効きません 😇
