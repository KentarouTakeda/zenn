---
title: "Laravel で HTTP Request をコンソールコマンドからディスパッチする"
emoji: "🌐"
type: "tech"
topics: ["php", "laravel", "console"]
published: true
---

## これは何？

`app/Http/Controllers/*` に置いたコントローラについて，以下の課題を考える。

| 実行の起点 | `PHP_SAPI` の値 | どのように実行されるか |
|:---|:---|:---|
| HTTP リクエスト | `mod_php` (Apache)<br>`fpm-fcgi` (php-fpm) | `public/index.php` に書かれたフレームワーク起動〜終了までの処理による |
| コマンドライン | `php-cli` | ？ |

通常 HTTP コントローラをその本来の用途以外で使うことは割けるべきであるが， Webhook 用のコントローラの動作確認のために使用するなどの場合には有用であると考えた。

## 実装

```
.
└── app/
    └── Console/
        └── Commands/
            └── Playground/
                ├── HandlesIncomingRequests.php
                └── IncomingRequestsHandler.php
```

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands\Playground;

use Illuminate\Contracts\Foundation\Application;
use Symfony\Component\Console\Output\OutputInterface;

trait HandlesIncomingRequests
{
    /**
     * @return Application
     * @noinspection PhpMissingReturnTypeInspection
     */
    abstract public function getLaravel();

    /**
     * @return OutputInterface
     * @noinspection PhpMissingReturnTypeInspection
     */
    abstract public function getOutput();

    /**
     * Content-Type: application/x-www-form-urlencoded のリクエストをハンドル
     * （GET や HEAD の場合には，クエリストリングを付加した場合と同等）
     *
     * @phpstan-param 'GET'|'HEAD'|'POST'|'PUT'|'PATCH'|'DELETE' $method
     * @noinspection PhpUnhandledExceptionInspection
     */
    public function handleQueryRequest(string $method, string $uri, string $query): void
    {
        $this->getLaravel()
            ->make(IncomingRequestHandler::class, ['output' => $this->getOutput()])
            ->handleQueryRequest($method, $uri, $query);
    }

    /**
     * application/json のリクエストをハンドル
     *
     * @phpstan-param 'POST'|'PUT'|'PATCH'|'DELETE' $method
     * @noinspection PhpUnhandledExceptionInspection
     */
    public function handleJsonRequest(string $method, string $uri, string $json): void
    {
        $this->getLaravel()
            ->make(IncomingRequestHandler::class, ['output' => $this->getOutput()])
            ->handleJsonRequest($method, $uri, $json);
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands\Playground;

use Illuminate\Contracts\Http\Kernel;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Symfony\Component\Console\Output\OutputInterface;

class IncomingRequestHandler
{
    public function __construct(
        private Kernel $kernel,
        private OutputInterface $output,
    ) {
    }

    /**
     * Content-Type: application/x-www-form-urlencoded のリクエストをハンドル
     * （GET や HEAD の場合には，クエリストリングを付加した場合と同等）
     *
     * @phpstan-param 'GET'|'HEAD'|'POST'|'PUT'|'PATCH'|'DELETE' $method
     */
    public function handleQueryRequest(string $method, string $uri, string $query): void
    {
        parse_str($query, $parsed);

        $this->handle(Request::create(
            uri: $uri,
            method: $method,
            parameters: $parsed,
            server: [
                'HTTP_ACCEPT' => 'application/json',
            ],
        ));
    }

    /**
     * Content-Type: application/json のリクエストをハンドル
     *
     * @phpstan-param 'POST'|'PUT'|'PATCH'|'DELETE' $method
     */
    public function handleJsonRequest(string $method, string $uri, string $json): void
    {
        $this->handle(Request::create(
            uri: $uri,
            method: $method,
            server: [
                'CONTENT_TYPE' => 'application/json',
                'HTTP_ACCEPT' => 'application/json',
            ],
            content: $json,
        ));
    }

    private function handle(Request $request): void
    {
        // カーネルでハンドル
        $response = $this->kernel->handle($request);

        // 中身を取り出す
        $content = $response instanceof JsonResponse
            ? $response->getData()
            : $response->getContent();

        // デバッグ用にコンソール出力
        $this->output->writeln((string)json_encode(
            $content,
            flags: JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES,
        ));

        // 登録された終了時のコールバックを実行
        $this->kernel->terminate($request, $response);
    }
}
```

## 使い方

Slack から Webhook としてリクエストされる Interactive Messages を捌く例

```php
.
└── app/
    └── Console/
        └── Commands/
            └── Playground/
                ├── Slack/
                │   └── Interactions/
                │       └── DispatchCommand.php
                ├── HandlesIncomingRequests.php
                └── IncomingRequestsHandler.php
```

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands\Playground\Slack\Interactions;

use App\Console\Commands\Playground\HandlesIncomingRequests;
use Illuminate\Console\Command;

class DispatchCommand extends Command
{
    use HandlesIncomingRequests;

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'playground:slack:interactions:dispatch
                            {json : JSON ペイロード引数}';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = '[動作確認] Slack の Webhook インタラクションを発生させる';

    public function handle(): void
    {
        $json = $this->argument('json');
        assert(is_string($json));

        $this->handleJsonRequest('POST', '/slack/interactions', $json);
    }
}
```
