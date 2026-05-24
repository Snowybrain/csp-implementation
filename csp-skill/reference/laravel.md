# CSP — Laravel / PHP (Blade)

Applies to: Laravel 5.x through 11.x with Blade templates.

---

## Middleware

**Location**: `app/Http/Middleware/CspMiddleware.php`

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\View;

class CspMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        if (!config('csp.enabled', true)) {
            View::share('cspNonce', '');
            return $next($request);
        }

        $nonce = base64_encode(random_bytes(16));
        View::share('cspNonce', $nonce);

        $response = $next($request);

        // Skip non-HTML responses (JSON, files, images)
        $ct = $response->headers->get('Content-Type', '');
        foreach (['application/json', 'application/octet-stream', 'image/', 'application/pdf'] as $skip) {
            if (str_contains($ct, $skip)) return $response;
        }

        $enforce = config('csp.enforce', false);
        $header  = $enforce ? 'Content-Security-Policy' : 'Content-Security-Policy-Report-Only';

        $response->headers->set($header, $this->buildPolicy($nonce, $enforce));

        // Report-To for modern browsers (complements report-uri)
        $response->headers->set('Report-To', json_encode([
            'group'     => 'csp-endpoint',
            'max_age'   => 86400,
            'endpoints' => [['url' => url('/csp-report')]],
        ]));

        return $response;
    }

    private function buildPolicy(string $nonce, bool $enforce): string
    {
        if (!$enforce) {
            return $this->reportOnlyPolicy($nonce);
        }
        return $this->enforcementPolicy($nonce);
    }

    private function reportOnlyPolicy(string $nonce): string
    {
        // Permissive — unsafe-inline included so nothing breaks yet
        // Replace CDN domains with the ones your project actually uses
        return implode('; ', [
            "default-src 'self'",
            "script-src 'self' 'nonce-{$nonce}' 'strict-dynamic' 'unsafe-inline' https://cdn.jsdelivr.net https://www.googletagmanager.com",
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://cdn.jsdelivr.net",
            "font-src 'self' data: https://fonts.gstatic.com",
            "img-src 'self' data:",
            "connect-src 'self'",
            "frame-src 'none'",
            "object-src 'none'",
            "base-uri 'self'",
            "form-action 'self'",
            "report-to csp-endpoint",
            "report-uri /csp-report",
        ]);
    }

    private function enforcementPolicy(string $nonce): string
    {
        return implode('; ', [
            "default-src 'self'",
            "script-src 'nonce-{$nonce}' 'strict-dynamic'",
            // style-src-elem controls <link> and <style> tags
            "style-src-elem 'self' 'nonce-{$nonce}' https://fonts.googleapis.com https://cdn.jsdelivr.net",
            // style-src-attr controls style="" — unsafe-inline needed until Bootstrap/DataTables migrated
            "style-src-attr 'unsafe-inline'",
            "font-src 'self' data: https://fonts.gstatic.com",
            "img-src 'self' data:",
            "connect-src 'self'",
            "frame-src 'none'",
            "frame-ancestors 'none'",
            "object-src 'none'",
            "base-uri 'self'",
            "form-action 'self'",
            ...(!app()->environment('local') ? ["upgrade-insecure-requests"] : []),
            "report-to csp-endpoint",
            "report-uri /csp-report",
        ]);
    }
}
```

## Register in Kernel

**Location**: `app/Http/Kernel.php` — add to the `web` group, after session middleware:

```php
'web' => [
    // ... existing middleware ...
    \App\Http\Middleware\VerifyCsrfToken::class,
    \App\Http\Middleware\CspMiddleware::class,  // ← add here
],
```

## Config file

**Location**: `config/csp.php`

```php
<?php
return [
    'enabled' => env('CSP_ENABLED', true),
    'enforce' => env('CSP_ENFORCE', false),  // false = Report-Only
];
```

## .env variables

```env
CSP_ENABLED=true
CSP_ENFORCE=false   # Start here. Switch to true only after days of clean Report-Only logs.
```

## Blade — using the nonce

```blade
{{-- Scripts: nonce on the <script> tag --}}
<script nonce="{{ $cspNonce }}" src="{{ asset('js/app.js') }}"></script>

{{-- Inline scripts (unavoidable): nonce on the block --}}
<script nonce="{{ $cspNonce }}">
    window.config = { locale: '{{ App::getLocale() }}' };
</script>

{{-- Better: use data-attribute to eliminate the inline script entirely --}}
<html data-locale="{{ App::getLocale() }}">
{{-- Then in JS: const locale = document.documentElement.dataset.locale --}}

{{-- Styles --}}
<link rel="stylesheet" href="{{ asset('css/app.css') }}">  {{-- no nonce needed on <link> with style-src-elem 'self' --}}
<style nonce="{{ $cspNonce }}"> /* only if truly unavoidable */ </style>
```

## CSP report endpoint

**Location**: `routes/web.php` — must be outside CSRF middleware:

```php
Route::post('csp-report', function (\Illuminate\Http\Request $request) {
    $body   = $request->getContent();
    $report = json_decode($body, true);
    $v      = $report['csp-report'] ?? $report ?? [];

    $entry = sprintf(
        "[%s] directive=%-30s blocked=%-50s doc=%s source=%s:%s ua=%s\n",
        now()->toIso8601String(),
        $v['violated-directive']  ?? '-',
        $v['blocked-uri']         ?? '-',
        $v['document-uri']        ?? '-',
        $v['source-file']         ?? '-',
        $v['line-number']         ?? '-',
        substr($request->userAgent() ?? '-', 0, 80)
    );

    file_put_contents(storage_path('logs/csp-violations.log'), $entry, FILE_APPEND | LOCK_EX);
    return response('', 204);
})->withoutMiddleware([\App\Http\Middleware\VerifyCsrfToken::class]);
```

Monitor: `tail -f storage/logs/csp-violations.log`

## Common issues in Laravel

| Symptom | Cause | Fix |
|---|---|---|
| Nonce not available in `@include` partial | `View::share` not called | Ensure middleware runs before view rendering |
| Google Analytics blocked | Inline `gtag()` snippet | Extract to `public/js/analytics.js` + load with `<script nonce>` |
| Bootstrap dropdowns broken | Bootstrap injects `style=` via JS | Keep `style-src-attr 'unsafe-inline'` |
| DataTables not rendering | DataTables injects inline styles | Same as above |
| Logout form broken | `onclick` on logout link | Replace with `<button type="submit">` inside the form |
| Queue/job views broken | Horizon/Telescope have own CSP | Exclude those routes from the middleware |
