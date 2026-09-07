# PageSpeed CI Baseline — Design Spec

Date: 2026-09-07
Status: Approved design, not yet implemented

## 1. Motivation

`laravel-seo-scanner` already integrates real Google PageSpeed Insights
(Lighthouse) data:

- `src/Support/PageSpeed.php` — thin HTTP client for the PSI API, cached per
  url+strategy.
- `src/Checks/PageSpeed/PerformanceScoreCheck.php`,
  `LcpCheck.php`, `ClsCheck.php` — checks that fail if the score/LCP/CLS
  don't meet a fixed threshold (`PerformanceScoreCheck::$expectedValue = 90`).
- `config/seo.php` `pagespeed.*` — opt-in via `SEO_PAGESPEED_API_KEY`; the
  three checks are excluded by default (`exclude_checks`).

Two things are missing that block using this for CI gating:

1. **No baseline.** Every page is judged against the same fixed threshold
   (90). Real sites — especially legacy ones — often can't hit 90 on every
   page, so there's no way to say "don't get worse than today" the way this
   repo's own CI already does for PHPStan (`phpstan.yml`, baseline-based).
2. **No failing exit code.** `SeoScan::handle()` always `return
   self::SUCCESS`, regardless of `$this->failed`. `SeoScanUrl` only fails if
   the URL itself was unreachable. There is currently no way for a CI step
   running `seo:scan` to actually fail the build on a check failure — of any
   kind, not just PageSpeed.

This spec covers both gaps, scoped to what's needed to make PageSpeed
regressions block a CI job. It does **not** touch this repo's own
`.github/workflows/*` — this package is a library with no live pages of its
own to scan; the baseline/CI mechanism is a feature for consumers to use in
their own apps' CI.

## 2. Goals

- A committable baseline file that snapshots the current PageSpeed score per
  URL.
- `PerformanceScoreCheck` fails only on a *regression* relative to that
  baseline (beyond a configurable tolerance), falling back to the existing
  fixed threshold for URLs not yet in the baseline.
- A general, opt-in way for `seo:scan` / `seo:scan-url` to exit non-zero
  when checks fail, so CI can act on it.
- README documentation showing a consumer how to wire this into GitHub
  Actions.

## 3. Non-goals

- `LcpCheck` and `ClsCheck` are **not** baselined in this pass — they keep
  their existing fixed thresholds, untouched. (Decided explicitly: score
  only, to avoid mixing units — points vs milliseconds vs a unitless CLS
  score — into one tolerance config.)
- No new mechanism for filtering *which* checks can fail a build — the
  existing `checks` / `exclude_checks` config in `config/seo.php` already
  lets a consumer scope a CI run to just the checks they care about (e.g. a
  CI-specific config running only `PerformanceScoreCheck`).
- No changes to this repo's own CI workflows.
- No UI/dashboard for baseline trends — this is a file-based, git-diffable
  mechanism, matching the PHPStan baseline pattern this repo already uses.

## 4. Components

### 4.1 `Support/PageSpeedBaseline.php` (new)

Thin read/write wrapper around the baseline JSON file, mirroring the style
of the existing `Support/PageSpeed.php`.

```php
namespace Backstage\Seo\Support;

class PageSpeedBaseline
{
    public static function path(): string
    {
        return config('seo.pagespeed.baseline_path') ?? base_path('seo-pagespeed-baseline.json');
    }

    /** Score baselined for a url, or null if not present in the baseline file. */
    public static function for(string $url): ?int
    {
        // Reads + caches (per-request, static array cache — no need for the
        // tagged cache store used by PageSpeed::for(), this is a local file
        // read) the baseline file, returns $data[$url]['score'] ?? null.
    }

    /**
     * Overwrite the baseline file with the given url => score map.
     *
     * @param array<string, int> $scores
     */
    public static function write(array $scores): void
    {
        // Writes {url: {score: int, checked_at: ISO8601 string}} sorted by
        // url key, JSON_PRETTY_PRINT, so diffs are small and readable.
    }
}
```

File format (`seo-pagespeed-baseline.json`):

```json
{
    "https://example.com/": {
        "score": 87,
        "checked_at": "2026-09-07T12:00:00+00:00"
    },
    "https://example.com/blog": {
        "score": 64,
        "checked_at": "2026-09-07T12:00:03+00:00"
    }
}
```

Keyed by the full resolved URL — the same identifier already used
everywhere else in the package (e.g. `--format=json` output, cache keys in
`PageSpeed::for()`).

### 4.2 Config additions (`config/seo.php`, inside the existing `pagespeed` array)

```php
'pagespeed' => [
    'api_key' => env('SEO_PAGESPEED_API_KEY'),
    'strategy' => 'mobile',
    'timeout' => 60,

    // Path to the baseline file written by `seo:pagespeed-baseline` and
    // read by PerformanceScoreCheck. Defaults to base_path() so it lives
    // in the consuming app's repo root and can be committed.
    'baseline_path' => env('SEO_PAGESPEED_BASELINE_PATH'),

    // How many points a page's score may drop below its baselined score
    // before PerformanceScoreCheck fails. PSI/Lighthouse lab scores are
    // inherently noisy (live network + CPU measurement), so an exact
    // comparison would be flaky in CI.
    'regression_tolerance' => 5,
],
```

### 4.3 `PerformanceScoreCheck` change

Replace the fixed comparison with a resolved effective floor:

```php
public function check(Response $response, Crawler $crawler): bool
{
    if (! PageSpeed::enabled()) { ... unchanged ... }

    $data = PageSpeed::for($this->url);
    if ($data === null || $data['score'] === null) { ... unchanged ... }

    $this->actualValue = $data['score'];

    $baseline = PageSpeedBaseline::for($this->url);
    $tolerance = (int) config('seo.pagespeed.regression_tolerance', 5);

    $floor = $baseline !== null
        ? max(0, $baseline - $tolerance)
        : $this->expectedValue;

    if ($this->actualValue >= $floor) {
        return true;
    }

    $this->failureReason = $baseline !== null
        ? __('failed.pagespeed.baseline_regression', [
            'actualValue' => $this->actualValue,
            'baseline' => $baseline,
            'tolerance' => $tolerance,
        ])
        : __('failed.pagespeed.performance_score', [
            'actualValue' => $this->actualValue,
            'expectedValue' => $this->expectedValue,
        ]);

    return false;
}
```

`$this->expectedValue` (used for reporting) is updated to `$floor` before
`setResult()` runs, so `toArray()` output (`expectedValue` field consumed by
`--format=json`) reflects whichever bar actually applied.

New translation string: `failed.pagespeed.baseline_regression` in each
locale file (`en`, `nl`, `fr`, `pt_BR`), alongside the existing
`failed.pagespeed.*` strings.

### 4.4 `seo:pagespeed-baseline` command (new)

```
seo:pagespeed-baseline {--force : Overwrite an existing baseline file without prompting}
```

Behavior:

1. Reuses the existing route/model discovery logic currently living as
   private methods on `SeoScan` (`getRoutes()`, and the model-iteration
   logic in `calculateScoreForModel`). This logic is extracted into a
   shared collaborator (e.g. `Services/ScanTargetResolver` or similar —
   exact shape decided during planning) so `SeoScan` and this new command
   both depend on it rather than duplicating URL discovery.
2. For each resolved URL, calls `PageSpeed::for($url)` directly — **not**
   the full check suite. Snapshotting a score doesn't need all 24 SEO
   checks to run, so this is both faster and avoids side effects (DB scan
   records, etc.) unrelated to its purpose.
3. Requires `PageSpeed::enabled()` — errors out with a clear message
   pointing at `SEO_PAGESPEED_API_KEY` if not configured, same as the
   checks do.
4. If a baseline file already exists and `--force` wasn't passed, prompts
   for confirmation before overwriting (interactive) or fails with a
   message telling the user to pass `--force` (non-interactive / CI).
5. Writes the file via `PageSpeedBaseline::write()`.
6. Prints a summary table (url, score) in console mode; `--format=json`
   is **not** needed here — this command's output is the file, not a
   report to parse — but the console table is worth keeping legible for
   a human running it locally before committing.

### 4.5 `--fail-on-failure` flag on `seo:scan` and `seo:scan-url`

- `SeoScan::handle()`: after computing `$this->failed`, if
  `$this->option('fail-on-failure')` and `$this->failed > 0`, return
  `self::FAILURE` instead of `self::SUCCESS`. Applies uniformly whether
  scanning routes, models, or both.
- `SeoScanUrl::handle()` (both `outputJson` and `outputConsole` paths): if
  the flag is set and `$score->getFailedChecks()->isNotEmpty()`, return
  `self::FAILURE`.
- Default behavior (flag omitted) is unchanged in both commands — this is
  purely opt-in, no breaking change for existing consumers or for this
  package's own test suite.
- Not supported with `--queue` on `SeoScan` in this pass: queued scans are
  asynchronous (`Bus::batch(...)->dispatch()`) and the command returns
  before results exist, so there's no failed-check count available at
  return time. `--fail-on-failure --queue` together should error out
  immediately with a clear message rather than silently no-op.

## 5. Data flow (end to end, consumer's app)

1. Developer runs `php artisan seo:pagespeed-baseline` locally, reviews the
   printed table, commits `seo-pagespeed-baseline.json`.
2. On every PR, CI runs `php artisan seo:scan --fail-on-failure` (optionally
   against a `config/seo.php` variant/env that narrows `checks` to just
   `PerformanceScoreCheck` if the consumer only wants to gate on
   PageSpeed).
3. `PerformanceScoreCheck` resolves each URL's floor from the baseline file
   (minus tolerance) or the fixed default if unbaselined.
4. Any regression beyond tolerance fails the check → `$this->failed > 0` →
   `seo:scan` exits non-zero → CI job fails.
5. When the developer intentionally improves (or accepts a regression on) a
   page, they re-run `seo:pagespeed-baseline --force` and commit the diff —
   same workflow as `phpstan --generate-baseline`.

## 6. Error handling

- `PageSpeedBaseline::for()` returns `null` (never throws) if the file is
  missing, unreadable, or the URL isn't a key — checks already handle a
  `null`/absent baseline by falling back to the fixed threshold.
- A malformed baseline file (invalid JSON) should log a warning once and be
  treated as "no baseline" rather than fatally erroring a scan — a broken
  baseline file shouldn't take down unrelated checks.
- `seo:pagespeed-baseline` without `SEO_PAGESPEED_API_KEY` configured fails
  fast with a message, same wording style as the existing
  `failed.pagespeed.not_configured` check failure.
- `--fail-on-failure --queue` together on `seo:scan` fails fast with a
  descriptive error (see 4.5).

## 7. Testing plan

- **`PageSpeedBaseline`**: write-then-read roundtrip; missing file returns
  null; malformed JSON returns null (and doesn't throw); path resolution
  respects `seo.pagespeed.baseline_path` config override.
- **`PerformanceScoreCheck`**: baseline absent → uses fixed threshold
  (existing behavior, regression-test it stays intact); baseline present
  and actual score within tolerance → passes; baseline present and actual
  score below `baseline - tolerance` → fails with the new failure reason;
  `expectedValue`/`actualValue` reported correctly in both branches (covers
  the `--format=json` consumer).
- **`seo:pagespeed-baseline` command**: writes expected file contents for a
  fixture set of routes; `--force` overwrite behavior; fails clearly when
  PageSpeed isn't enabled; fails clearly (or prompts) on pre-existing file
  without `--force`.
- **`--fail-on-failure`**: feature tests for both `seo:scan` and
  `seo:scan-url` asserting exit code 0 vs 1 with/without the flag, for both
  a fully-passing and a failing scan; `--fail-on-failure --queue` errors
  immediately.

## 8. Open items for the implementation plan

- Exact shape/name of the extracted route+model discovery collaborator
  (§4.4.1) — `SeoScan` currently has this logic as private methods; needs a
  home both commands can share without over-abstracting.
- Whether `seo:pagespeed-baseline` should also support scanning a single
  ad-hoc URL (mirroring `seo:scan-url`) or stay scoped to configured
  routes/models only. Leaning toward configured routes/models only for v1,
  since a baseline is meaningless for a URL that isn't part of the site's
  regular gated set — but worth confirming during planning.
