---
title: "Github Actions を定刻に実行する方法"
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions"]
published: true
publication_name: "no4_dev"
---

## `schedule` トリガーを用いた方法 👎

Github Actions を定刻に実行する方法としては、`schedule` トリガーを利用する方法が一般的かと思います。

```yaml:.github/workflows/scheduled.yaml
on:
  schedule:
    # * is a special character in YAML so you have to quote this string
    - cron: "30 5,17 * * *"
```

https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule

:::message
ただし、`schedule` トリガーを用いた方法は、公式ドキュメントにもあるように遅延する場合があります。

> Note: The schedule event can be delayed during periods of high loads of GitHub Actions workflow runs. High load times include the start of every hour. To decrease the chance of delay, schedule your workflow to run at a different time of the hour.

私は、午前 2:00 に指定しましたが、**毎回 20 分ほど**遅延しました 😢

:::

定刻に実行できるような Workflow も探してみましたが、GithubActions を実行したまま待機するなど、力技のものしか見つかりませんでしたので、外部サービスと連携して実現することにしました。

## `repository_dispatch` トリガーを用いた方法 👍

こちらは Github の Webhook を外部から呼び出した際に発生するイベントです。

```yaml:.github/workflows/webhooks.yaml
on:
  repository_dispatch:
    types: [on-demand-test]
```

https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch

### cron-job

無料で利用でき、任意のタイミングで http リクエストを送信できるサービスです。

https://cron-job.org/en/

こちらのサービスから Github の Webhook へ http リクエストを送信します。

サクッと登録して、Cronjob を作っていきます。

![cron-job-1.png](/images/14b295b8dafbfd/cron-job-1.png)

- Title: `<任意>`
- URL: `https://api.github.com/repos/<repository-owner>/<repository-name>/dispatches`
- Execution schedule: `<任意>`

![cron-job-2.png](/images/14b295b8dafbfd/cron-job-2.png)

- Headers
  - Accept : `application/vnd.github+json`
  - Authorization : `Bearer <personal-access-token>`
  - X-GitHub-Api-Version : `2022-11-28`
- Advanced
  - Time zone : `Asia/Tokyo`
  - Request method : `POST`
  - Request body: `{"event_type":"on-demand-test"}`

:::message
ここで利用する `personal-access-token` は以下の条件を満たす必要があります。

> This endpoint requires write access to the repository by providing either:
>
> - Personal access tokens with repo scope. For more information, see "Creating a personal access token for the command line" in the GitHub Help documentation.
> - GitHub Apps with both metadata:read and contents:read&write permissions.

:::

:::message
`Request body` は `event_type`は、`repository_dispatch`で指定した値と一致する必要があります。
また、`event_type`以外のパラメータも渡すことができます

```json
{
  "event_type": "test_result",
  "client_payload": {
    "passed": false,
    "message": "Error: timeout"
  }
}
```

:::

スケジュールには、毎日/月/年だけでなく、曜日指定もできるので、日/月次処理だけでなく、週次処理も実行できそうです 💡
