# telebot

A reusable, typed **asynchronous Telegram Bot API refrain for Encore**. Its
public futures perform nonblocking DNS/TCP/TLS/HTTP through `std::http::HTTPAsyncClient`.
Telegram models, JSON codecs and method futures are implemented in Encore.

`telebot` is an independent library. Its dependencies are `std` and `json`;
it does not import JevJob or any of its application packages. The library accepts
the token and HTTP client from its caller. It does not read JevJob configuration
or start the application.

The package directory can be used outside this repository. From that directory,
run `encore sync`, `encore check` and `encore build`. JevJob consumes the local
package through `path@packages/telebot`; another application can use its own path
dependency or the published index package.

The initial platform is Linux x86_64 with the patched local Encore 0.1.6 build,
std >=0.0.5, OpenSSL and c-ares >=1.34 development packages. The Telegram package
is published as `telebot` in the local Encore registry. Add
`index@telebot@^0.1.0` to a consumer's dependencies. This repository uses a local
path dependency for development; std and json come from the registry.

## Minimal use

```encore
import std::http::HTTPAsyncClient
import std::future::block_on
import core::result::Result
import telebot::{BotClient, BotError, User, GetMeFuture}

// Inside a scope that handles constructor errors:
// let http = HTTPAsyncClient::new()?
// with http as http {
//     let bot = BotClient::new(http, token)?
//     let user = block_on[Result[User, BotError], GetMeFuture](bot.get_me())?
// }
```

Method results are concrete, single-consumer futures implementing Encore's
`Future[Result[T, BotError]]` protocol, so callers can also `await` them inside
their own async workflow. Execute those workflows through `std::future::block_on`,
with explicit generic arguments as required by the current compiler.
The HTTP client owns its reactor; close it explicitly or use a with-context.
The package starts no polling loop and retains no update offset.

| Method | Result | Future type |
| --- | --- | --- |
| `get_me()` | `User` | `GetMeFuture` |
| `get_updates(GetUpdates)` | `Vec[Update]` | `GetUpdatesFuture` |
| `send_message(SendMessage)` | `Message` | `SendMessageFuture` |
| `edit_message_text(EditMessageText)` | `Message` | `EditMessageTextFuture` |
| `answer_callback_query(AnswerCallbackQuery)` | `bool` | `AnswerCallbackQueryFuture` |
| `get_file(file_id)` | `File` | `GetFileFuture` |
| `download_file(File)` | `DownloadedFile` | `DownloadFileFuture` |

The [echo example](../../examples/echo/src/main.enq) includes long polling,
messages and inline callback buttons. Start it from `examples/echo` with
`TELEGRAM_BOT_TOKEN` set, then `encore run`. `/quit` ends this private demo and
closes the HTTP client. The example does not implement application authorization.
`encore run -- --check` only verifies `getMe` and exits; it sends no messages.
`TELEGRAM_API_BASE` and `TELEGRAM_CA_FILE` configure an optional local server and
test CA. The repository fixture exercises this independent consumer too.

## Configuration and semantics

- `BotConfig` configures API/file endpoint prefixes, HTTP settings, and file
  limits. Production endpoints default to `https://api.telegram.org`.
- Default long polling is 30 seconds, with a 40-second overall deadline;
  longer polling requests automatically extend that deadline. Other API calls
  default to 15 seconds. Downloads default to 60 seconds and 20 MiB.
- `BotClient::with_cancellation(token)` applies an HTTP client cancellation token to
  API and download futures. Individual clients can use different tokens.
- ID values use exact signed `i64` parsing, never floating-point conversion.
  Missing optional fields use `Option`; unknown JSON fields/events remain in
  `raw`. Incorrect types in known fields fail decoding.
- Only callback and URL inline buttons are currently supported. Editing targets
  `chat_id`/`message_id`; inline-message editing is outside v0.1.
- `BotError` distinguishes configuration, transport, HTTP, protocol, API and
  file-limit errors. API errors preserve `api_code`, `retry_after` and
  `migrate_to_chat_id`. Descriptions redact the exact token.
- No automatic retries, offset advancement, webhook removal or message splitting
  occur. A caller decides whether a failed send is safe to repeat.
- `download_file` requires the `file_path` returned by `get_file`, validates that
  path, enforces limits while receiving data, and preserves arbitrary bytes in
  `DownloadedFile.bytes`. `str` is the current Encore binary buffer carrier;
  do not decode or normalize those bytes before writing a PDF.
- Token-bearing requests never follow redirects. Do not log `BotClient`, request
  values, future internals or environment credentials.

## Tests

Run `encore test` in this directory. The standalone network test uses
`JEV_TEST_URL` and optional `JEV_TEST_CA` to select a local Bot API fixture,
including binary files and API errors. It skips network IO when the URL is
absent. A real Telegram token is not required. The repository's
`tests/http_integration` fixture also runs standard-library transport tests
and the echo consumer over HTTP and HTTPS.

Reference: [Telegram Bot API](https://core.telegram.org/bots/api).
