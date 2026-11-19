# Signature HQ PHP SDK

Official PHP client for the [Signature HQ](https://signaturehq.com.au) e-signature API (v1).

- PHP 8.2+, strict types, immutable (`readonly`) response objects
- PSR-18 HTTP client: Guzzle by default, or bring your own
- Typed exceptions for every API error, automatic idempotency keys, safe retries with backoff
- Webhook signature verification

> Not published yet. The package name `signaturehq/signaturehq-php` and the default base URL
> `https://app.signaturehq.com.au/api/v1` are placeholders until the first release.

## Install

```bash
composer require signaturehq/signaturehq-php
```

## Quickstart

```php
use SignatureHQ\Client;

$shq = new Client(getenv('SHQ_API_KEY'));      // "shq_..." live, "shq_test_..." test mode

$envelope = $shq->envelopes->create(['title' => 'Lease agreement', 'signing_order' => 'parallel']);
$document = $shq->documents->upload($envelope->id, '/path/to/lease.pdf');
$signer = $shq->recipients->add($envelope->id, ['name' => 'Jane Citizen', 'email' => 'jane@example.com', 'role' => 'signer']);

// Positions are percentages of the page. set() replaces the whole field set.
$shq->fields->set($envelope->id, [[
    'document' => $document->id, 'recipient' => $signer->id, 'type' => 'signature',
    'page' => 1, 'x' => 10, 'y' => 80, 'width' => 30, 'height' => 6, 'required' => true,
]]);

$envelope = $shq->envelopes->send($envelope->id);

// Later, once $envelope->sealed is true:
$shq->envelopes->saveSealedDocument($envelope->id, '/path/to/signed.pdf');
```

`$shq->isTestKey()` / `$shq->mode()` (`"test"` or `"live"`) tell you which kind of key you configured. Test envelopes
are not legally binding, but their invitations are really emailed: use addresses you control. `$envelope->livemode`
is false for test envelopes.

### Every endpoint

| Call                                                                                                                | Endpoint                                  |
| ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `envelopes->list(['status' => 'sent', 'per_page' => 50, 'page' => 1])`                                              | `GET /envelopes`                          |
| `envelopes->all([...])`                                                                                             | every page, lazily                        |
| `envelopes->create([...], ?$idempotencyKey)`                                                                        | `POST /envelopes`                         |
| `envelopes->retrieve($id)`                                                                                          | `GET /envelopes/{id}`                     |
| `envelopes->update($id, [...])`                                                                                     | `PATCH /envelopes/{id}`                   |
| `envelopes->delete($id)`                                                                                            | `DELETE /envelopes/{id}` (drafts)         |
| `envelopes->send($id, ?$idempotencyKey)`                                                                            | `POST /envelopes/{id}/send`               |
| `envelopes->void($id, $reason)`                                                                                     | `POST /envelopes/{id}/void`               |
| `envelopes->downloadSealedDocument($id)` / `saveSealedDocument($id, $path)`                                         | `GET /envelopes/{id}/sealed-document`     |
| `documents->upload($envelopeId, $pathOrStream, ?$filename)`, `uploadFromPath`, `uploadFromStream`, `uploadContents` | `POST /envelopes/{id}/documents`          |
| `recipients->add($envelopeId, [...])`                                                                               | `POST /envelopes/{id}/recipients`         |
| `recipients->update($envelopeId, $recipientId, [...])`                                                              | `PATCH /envelopes/{id}/recipients/{rid}`  |
| `recipients->remove($envelopeId, $recipientId)`                                                                     | `DELETE /envelopes/{id}/recipients/{rid}` |
| `fields->set($envelopeId, [...])`                                                                                   | `PUT /envelopes/{id}/fields`              |
| `webhooks->list()`, `create([...])`, `retrieve($id)`, `update($id, [...])`, `delete($id)`                           | `/webhooks`                               |
| `webhooks->rotateSecret($id)`                                                                                       | `POST /webhooks/{id}/rotate-secret`       |
| `webhooks->ping($id)`                                                                                               | `POST /webhooks/{id}/ping`                |
| `webhooks->deliveries($id, [...])`, `allDeliveries($id)`                                                            | `GET /webhooks/{id}/deliveries`           |

Envelope create/update accept `title`, `message`, `signing_order` (`parallel`/`sequential`), `expires_on`
(`YYYY-MM-DD`, Australia/Sydney), `reminder_interval_days` (1-30 or null) and `append_certificate` (bool).
Recipients accept `verification_method` (`email_link` by default, `email_otp`, or `sms_otp` with `phone`). Radio and
dropdown fields take `options` (2 to 20 labels).
Every response object keeps the full payload in `->raw`, so fields added to the API later are readable before the
SDK knows them.

## Configuration

```php
$shq = new Client(
    getenv('SHQ_API_KEY'),
    baseUrl: 'https://app.signaturehq.com.au/api/v1', // https only (http for localhost)
    timeout: 60.0,            // seconds per attempt (default Guzzle client)
    connectTimeout: 10.0,
    maxRetries: 2,            // 0 disables retries
    httpClient: $psr18Client, // optional; then also requestFactory/streamFactory if not Guzzle's
    appInfo: 'acme-crm/2.1',  // appended to the User-Agent
    debug: fn (array $event) => error_log(json_encode($event)), // Authorization redacted, no bodies
);
```

The API key is never put in a URL, never logged, hidden from `var_dump`/`print_r` (not `var_export`), never chained
into exceptions, and the client refuses to be serialized. The default client never follows redirects (the key only ever
goes to the configured host): if you pass your own PSR-18 client, disable redirects on it too.

Sealed PDFs are streamed from the socket by the default client: `$timeout` is then the longest pause between reads,
not a limit on the whole download.

## Errors

Every error extends `SignatureHQ\Exception\SignatureHQException`. API errors (`ApiException`) carry `status`,
`type`, `getMessage()` and field `errors`:

| Exception                       | When                                                            |
| ------------------------------- | --------------------------------------------------------------- |
| `AuthenticationException`       | 401: invalid, expired or revoked key                            |
| `PermissionException`           | 403: missing ability or permission                              |
| `NotFoundException`             | 404 (envelopes of other organisations are always 404)           |
| `ConflictException`             | 409: e.g. editing an envelope that was already sent             |
| `IdempotencyConflictException`  | 409: the same key is still being processed                      |
| `ValidationException`           | 422: `fieldErrors('email')`                                     |
| `IdempotencyKeyReusedException` | 422: the key was used for a different request                   |
| `RateLimitException`            | 429: `retryAfter` seconds                                       |
| `ServerException`               | 5xx                                                             |
| `InvalidRequestException`       | 400 and other 4xx                                               |
| `ConnectionException`           | network failure or timeout                                      |
| `InvalidArgumentException`      | rejected locally before sending (bad id, bad key, missing file) |

```php
try {
    $shq->recipients->add($envelopeId, ['name' => 'Jane', 'email' => 'nope', 'role' => 'signer']);
} catch (SignatureHQ\Exception\ValidationException $e) {
    $messages = $e->fieldErrors('email');
}
```

## Idempotency and retries

`envelopes->create()` and `envelopes->send()` always send an `Idempotency-Key` (a random UUID unless you pass one) and
reuse it for their retries, so a retry does not create a second envelope or send twice once the server has answered. Pass your own key (8-255 characters:
letters, digits, `-`, `_`, `:`, `.`) to stay safe across process restarts:

```php
$shq->envelopes->create(['title' => 'Order 1234'], idempotencyKey: 'order-1234-envelope');
```

Retries (default 2, exponential backoff with jitter, at most 8 s between attempts):

- network errors and 5xx: only for `GET`, `PUT`, `DELETE` and the keyed create/send;
- 429: always (the rate limiter refuses before any work is done), waiting for `Retry-After` up to 60 s
  (`maxRetryAfter`);
- 409 `idempotency_conflict`: for keyed requests.

Other `POST`/`PATCH` calls (void, add recipient, upload, webhook create, rotate secret, updates) are never retried after
they may have reached the server. A `DELETE` retried after a network error, timeout or 5xx that then gets 404 (the first attempt went through) counts
as success; after a 429 a 404 is reported as usual.

When retries run out, the error carries the key that was sent (`idempotencyKey`): retry later with that same key, never a
new one, or you may create a second envelope. Keys protect against duplicates only for 24 hours.

The server stores 4xx responses for a key too. If `send` fails with a 422 because the draft is incomplete, fix the
draft and send again with a new key: reusing the old key replays the same 422 for 24 hours.

## Pagination

```php
$page = $shq->envelopes->list(['status' => 'completed', 'per_page' => 100]);
$page->data; $page->total; $page->hasMore(); $next = $page->nextPage();

foreach ($shq->envelopes->all(['status' => 'completed']) as $envelope) {
    // every page, fetched lazily; ids already seen are skipped if new envelopes shift the pages
}
```

## Webhooks

Verify every delivery against the **raw** body before using it:

```php
use SignatureHQ\Webhook;
use SignatureHQ\Exception\SignatureVerificationException;

$payload = file_get_contents('php://input');
try {
    $event = Webhook::constructEvent($payload, $_SERVER['HTTP_SHQ_SIGNATURE'] ?? null, getenv('SHQ_WEBHOOK_SECRET'));
} catch (SignatureVerificationException) {
    http_response_code(400);
    exit;
}
http_response_code(204); // answer fast, then process asynchronously, de-duplicating on $event->id (SHQ-Event-Id)

if ($event->type === 'envelope.sealed') {
    $shq->envelopes->saveSealedDocument($event->envelopeId(), "/secure/{$event->envelopeId()}.pdf");
}
```

- `Webhook::verifySignature($payload, $header, $secret)` returns a bool instead.
- The timestamp must be within 5 minutes (`tolerance:` to change it).
- Rotating a secret switches deliveries to the new one immediately: pass both while you deploy,
  `[$newSecret, $oldSecret]`.
- `Webhook::signatureHeader($payload, $secret)` builds a header for your own tests.

## Development

```bash
composer install
vendor/bin/phpunit --testsuite unit
vendor/bin/phpstan analyse
```

The integration smoke test runs against a local SHQ e2e server: from the repository root inside DDEV,
`E2E_PORT=8024 E2E_DB=e2e_k node sdks/integration/run.mjs php`.
