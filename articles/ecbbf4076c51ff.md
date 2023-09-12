---
title: "【React/TypeScript】Dialogタグを使ってコンポーネントを作ってみる ver.2022.09"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript", "dialog", "modal"]
published: true
publication_name: "no4_dev"
---

## はじめに

Modal/Dialog は実際のプロジェクトでもよく使うと思います。
有名な React UI ライブラリの[Chakra UI](https://chakra-ui.com/) や [mui](https://mui.com/) にも含まれており大変便利なのですが、
プロジェクト途中からこういったライブラリを導入するのはハードルが非常に高くなってしまいます。
一方、もし自作する場合、普段何気なく使っている Modal/Dialog にも考慮しないといけない仕様も多くわりと大変です。
今回は、自分の中での整理/知識のアップデートもかねて、Dialog を作ってみようと思います。

## 実装に入る前に

今回ポイントとなる情報を整理します。

### Dialog タグ

https://developer.mozilla.org/ja/docs/Web/HTML/Element/dialog

少し前に追加されたタグで、最近 Safari が対応したことで主なモダンブラウザ全般で利用できるようになりました。

https://caniuse.com/dialog

表示/非表示を切り替える `showModal` / `close` メソッド、表示状態を表す`open` 属性、Overlay を表現するための`backdrop` 疑似要素など、Dialog を作るうえであると便利な機能が備わっているタグです。

また z-index の影響を影響うけない [top-layer](https://fullscreen.spec.whatwg.org/#new-stacking-layer) というところに描画されるため、後ろのコンテンツの z-index が高くても問題ありません。

### Focus Trap

`Tab`や`Shift + Tab`でフォーカスを移動した際、Dialog の後ろのコンテンツにフォーカスが移動しないようにフォーカス移動を制御する必要があります。
有名なのは以下のライブラリですが、Dialog タグを使うとブラウザが標準で`Focus Trap`を行ってくれるようなので、今回はライブラリは使いません。
※ サポートブラウザの関係で Dialog タグが利用できない場合は、以下のライブラリ(または類似のライブラリ)を導入する必要があります。

https://github.com/focus-trap/focus-trap

### スクロールの連鎖

https://coliss.com/articles/build-websites/operation/css/prevent-scroll-chaining-overscroll-behavior.html

上記の記事からの引用となりますが、

> スクロールの連鎖（スクロールチェーン）とは、ページ上にスクロールするコンテンツがあり、そのコンテンツをスクロールして終点に到達するとメインのコンテンツもスクロールしてしまう現象です。

という現象であり、Dialog を実装する上で避けて通れない課題の一つです。

記事中にもあるように[overscroll-behavior](https://developer.mozilla.org/ja/docs/Web/CSS/overscroll-behavior) プロパティは非常に有用なのですが、このプロパティを設定した要素にスクロールが発生している状態でないと効きません。
つまり、Dialog の高さが小さい場合は背景がスクロールされてしまいます。

現状、この問題を CSS だけで解決するのは難しいようなので、Chakra UI でも利用されている以下のモジュールを利用します。

https://www.npmjs.com/package/react-remove-scroll

## 実装

### 1. Dialog を作る

まず、シンプルに開閉できる Dialog を作ります。

```ts:src/Dialog/Dialog.tsx
import { useRef, useEffect } from "react";

import classes from "./Dialog.module.css";

type Props = {
  isOpen?: boolean;
  children: React.ReactNode | React.ReactNodeArray;
};

export const Dialog: React.FC<Props> = ({
  isOpen = false,
  children,
}:Props): React.ReactElement => {
  const dialogRef = useRef<HTMLDialogElement>(null);

  useEffect((): void => {
    const dialogElement = dialogRef.current;
    if (!dialogElement) {
      return;
    }
    if (isOpen) {
      if (dialogElement.hasAttribute("open")) {
        return;
      }
      dialogElement.showModal();
    } else {
      if (!dialogElement.hasAttribute("open")) {
        return;
      }
      dialogElement.close();
    }
  }, [isOpen]);

  return (
      <dialog
        className={classes["dialog"]}
        ref={dialogRef}
      >
        <div className={classes["content"]} >
          {children}
        </div>
      </dialog>
  );
};

```

スタイルは今回はこだわらないので、ほぼほぼデフォルトのままです。

```css:src/Dialog/Dialog.module.css
/**
 * ::backdrop => Dialogの背景(Overlay)の疑似要素
 * :modal     => DialogタグをshowModal()で表示した際の疑似セレクタ
 */

.dialog::backdrop {
  opacity: 0;
  background: rgba(0, 0, 0, 0.5);
}

.dialog:modal::backdrop {
  opacity: 1;
  animation: fadein 0.15s ease-in; /** backdropに表示アニメーションを付与 */
}

@keyframes fadein {
  0% {
    opacity: 0;
  }
  100% {
    opacity: 1;
  }
}

```

次に、backdrop 部分をクリックした場合、Dialog を閉じるよう修正します。
ただし、疑似要素である `::backdrop` はイベントは拾えないため、親の`.dialog`の onClick を利用します。
※そのままだと dialog の中(`.content`内)のクリックイベントも伝搬してしまうため、click イベントの伝搬を止める必要があります。

```diff ts:src/Dialog/Dialog.tsx
- import { useRef, useEffect } from "react";
+ import { useCallback, useEffect, useRef } from "react";

import classes from "./Dialog.module.css";

type Props = {
  isOpen?: boolean;
  children: React.ReactNode | React.ReactNodeArray;
+  onClose?: VoidFunction;
};

export const Dialog: React.FC<Props> = ({
  isOpen = false,
  children,
+  onClose,
}): React.ReactElement | null => {
  const dialogRef = useRef<HTMLDialogElement>(null);

  useEffect((): void => {
    const dialogElement = dialogRef.current;
    if (!dialogElement) {
      return;
    }
    if (isOpen) {
      if (dialogElement.hasAttribute("open")) {
        return;
      }
      dialogElement.showModal();
    } else {
      if (!dialogElement.hasAttribute("open")) {
        return;
      }
      dialogElement.close();
    }
  }, [isOpen]);
+
+  const handleClickDialog = useCallback(
+    (): void => {
+      onClose?.();
+    },
+    [onClose]
+  );
+
+  const handleClickContent = useCallback(
+    (event: React.MouseEvent<HTMLDivElement>): void => {
+      // clickイベントの伝搬を止める。
+      event.stopPropagation();
+    },
+    []
+  );

  return (
    <dialog
      className={classes["dialog"]}
      ref={dialogRef}
+      onClick={handleClickDialog}
    >
-      <div className={classes["content"]}>
+      <div className={classes["content"]} onClick={handleClickContent}>
        {children}
      </div>
    </dialog>
  );
};

```

Dialog 内の Padding を.content 内に移します。
※そのままだと、余白部分が.dialog のクリック判定になってしまい、Dialog が閉じてしまいます。

```diff css:src/Dialog/Dialog.module.css
/**
 * ::backdrop => Dialogの背景(Overlay)の疑似要素
 * :modal     => DialogタグをshowModal()で表示した際の疑似セレクタ
 */

+ .dialog {
+   padding: 0;
+ }

.dialog::backdrop {
  opacity: 0;
  background: rgba(0, 0, 0, 0.5);
}

.dialog:modal::backdrop {
  opacity: 1;
  animation: fadein 0.15s ease-in; /** backdropに表示アニメーションを付与 */
}

@keyframes fadein {
  0% {
    opacity: 0;
  }
  100% {
    opacity: 1;
  }
}

+ .content {
+   padding: 1em;
+ }

```

最後に、スクロールが連鎖しないように、修正します。

```diff ts:src/Dialog/Dialog.tsx
import { useCallback, useEffect, useRef } from "react";
+ import { RemoveScroll } from "react-remove-scroll";

import classes from "./Dialog.module.css";

type Props = {
  isOpen: boolean;
  children: React.ReactNode | React.ReactNodeArray;
 onClose: VoidFunction;
};

export const Dialog: React.FC<Props> = ({
  isOpen,
  children,
 onClose,
}): React.ReactElement | null => {
  const dialogRef = useRef<HTMLDialogElement>(null);

  useEffect((): void => {
    const dialogElement = dialogRef.current;
    if (!dialogElement) {
      return;
    }
    if (isOpen) {
      if (dialogElement.hasAttribute("open")) {
        return;
      }
      dialogElement.showModal();
    } else {
      if (!dialogElement.hasAttribute("open")) {
        return;
      }
      dialogElement.close();
    }
  }, [isOpen]);

  const handleClickDialog = useCallback(
   (): void => {
      onClose();
    },
    [onClose]
  );

  const handleClickContent = useCallback(
    (event: React.MouseEvent<HTMLDivElement>): void => {
      // clickイベントの伝搬を止める。
      event.stopPropagation();
    },
    []
  );

  return (
+     <RemoveScroll removeScrollBar enabled={isOpen}>
      <dialog
        className={classes["dialog"]}
        ref={dialogRef}
       onClick={handleClickDialog}
      >
       <div className={classes["content"]} onClick={handleClickContent}>
          {children}
        </div>
      </dialog>
+     </RemoveScroll>
  );
};

```

ついでにエントリポイントも作っておきます。

```ts:src/Dialog/index.ts
export * from "./Dialog";

```

#### 2. Dialog を render hook でラップする

特別なことは不要で、通常の render hooks を作ります。

:::message

render hooks パターンとは

@uhyo さんの記事で知った手法です。
カスタムフックで`Component`と`その状態を変更する関数`を返すことにより、状態変更時のロジックを隠蔽できるメリットがああります。

https://engineering.linecorp.com/ja/blog/line-securities-frontend-3/
:::

```ts:src/Dialog/useDialog.tsx
import { useCallback, useState } from "react";

import { Dialog as Component } from "./Dialog";

type Props = Omit<
  Parameters<typeof Component>[0],
  "isOpen" | "onClose" | "rootElement"
>;

type Result = {
  open: VoidFunction;
  close: VoidFunction;
  Dialog: React.FC<Props>;
};

export const useDialog = (): Result => {
  const [isOpen, setOpen] = useState<boolean>(false);

  const open: VoidFunction = useCallback((): void => {
    setOpen(true);
  }, []);

  const close: VoidFunction = useCallback((): void => {
    setOpen(false);
  }, []);

  const Dialog: React.FC<Props> = useCallback(
    (props: Props): React.ReactElement => {
      return <Component isOpen={isOpen} onClose={close} {...props} />;
    },
    [close, isOpen]
  );

  return { open, close, Dialog };
};

```

エントリポイント追記します。

```diff ts:src/Dialog/index.ts
export * from "./Dialog";
+ export * from "./useDialog";

```

以上で実装は完了です。

### 利用方法

利用する際は以下のように render hooks 経由で呼び出します。

```ts
import { useState } from "react";
import { useDialog, Dialog } from "./Dialog";

export const Demo:React.FC => ():React.ReactElement {
  const [open, setOpen] = useState<boolean>(false);

  const { Dialog: RenderDialog, open: openDialog, close: closeDialog } = useDialog();

  return (
    <>
      <section>
        <button type="button" onClick={() => setOpen(true)}>
          open dialog
        </button>
        <Dialog isOpen={open} onClose={() => setOpen(false)}>
          <header>
            <h2>タイトル</h2>
          </header>
          <section>コンテンツ</section>
          <footer>
            <button type="button" onClick={() => setOpen(false)}>
              close
            </button>
          </footer>
        </Dialog>
      </section>

      <section>
        <button type="button" onClick={openDialog}>
          open dialog
        </button>
        <RenderDialog>
          <header>
            <h2>タイトル</h2>
          </header>
          <section>コンテンツ</section>
          <footer>
            <button type="button" onClick={closeDialog}>
              close
            </button>
          </footer>
        </RenderDialog>
      </section>
    </>
  );
}

```

### 完成したもの

実際に動かせるサンプルは以下に置いてあります。

https://codesandbox.io/p/sandbox/react-dialog-xl7tmz?file=/src/Dialog/index.ts

### TODO

気が向いたときに追記したいと思っています。

- a11y にも対応する。
- もう一層 render hooks でラップし、以下の記事のように Promise を使う。
  - https://zenn.dev/yumemi_inc/articles/f6ec17edf13670
