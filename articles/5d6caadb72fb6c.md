---
title: "【JavaScript/TypeScript】Array.reduceでもasync/awaitをを使う"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "typescript", "promise"]
published: true
publication_name: "no4_dev"
---

Array.map で async/await を使う方法は調べるとすぐに出てきますが、Array.reduce で async/await を使う方法が調べても出てこなかったため、忘備録です。

```tsx
/**
 * 非同期処理
 */
const asyncFunc = (id: number): Promise<string> => {
  return new Promise<string>((resolve: (value: string) => void): void => {
    console.count("start Promise");
    setTimeout((): void => {
      console.count("end Promise");
      resolve(`ID:${id}`);
    }, 100);
  });
};

/**
 * reduce変換後の型
 */
type Result = { [key: number]: string };

const loopFunc = async (): Promise<void> => {
  // 初期値
  const list: number[] = Array.from(Array(5), (_, i) => i);
  console.info(list);
  // (5) [0, 1, 2, 3, 4]

  // loop処理
  const result = list.reduce<Promise<Result>>(
    async (prev: Promise<Result>, value: number): Promise<Result> => {
      return {
        ...(await prev), // prevのPromiseを解決する
        [value]: await asyncFunc(value),
      };
    },
    // 初期値
    Promise.resolve<Result>({})
  );

  // 結果確認
  console.info(await result);
  // {0: "ID:0", 1: "ID:1", 2: "ID:2", 3: "ID:3", 4: "ID:4"}
};

loopFunc();
```

Array.map で Promise 解決してから、Array.reduce を実行するという方法があることも認識はしていますが、今回は諸事情で Array.reduce でやりたかったんです....

実際に動かせるサンプルコードはこちら ↓
https://codesandbox.io/s/async-array-reduce-hv5uvb?file=/src/index.ts
