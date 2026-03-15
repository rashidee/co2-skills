# Security Patterns — Keycloak OAuth2 Client with Laravel Socialite

This reference describes the complete security architecture for the spec. Include all of
this content in Section 6 of the generated specification.

---

## Architecture Overview

The application operates as an OAuth2 Client using Authorization Code flow with OIDC.
Keycloak is the external identity provider that handles login UI, user registration,
and role management. The application's responsibilities are:

1. Redirect unauthenticated users to Keycloak login page
2. Handle the OAuth2 callback and establish an HTTP session
3. Extract roles from OIDC ID token claims
4. Enforce role-based access control on routes

The application uses server-side sessions (Laravel's default). After successful OAuth2
login, the authenticated user is stored in the session. User identity and roles are
read from the OIDC ID token claims.

The flow:
```
Browser → GET /protected-page → Auth Middleware (not authenticated)
    → 302 Redirect to Keycloak /auth?response_type=code&client_id=...
    → User logs in at Keycloak
    → Keycloak redirects back with authorization code
    → Laravel exchanges code for tokens (ID token + access token) via Socialite
    → KeycloakRoleMapper extracts roles from ID token
    → Session created, User stored as authenticated principal
    → 302 Redirect to original page
```

### Configuration in .env

```env
KEYCLOAK_CLIENT_ID={{KEYCLOAK_CLIENT_ID}}
KEYCLOAK_CLIENT_SECRET={{KEYCLOAK_CLIENT_SECRET}}
KEYCLOAK_REALM={{KEYCLOAK_REALM}}
KEYCLOAK_BASE_URL={{KEYCLOAK_BASE_URL}}
KEYCLOAK_REDIRECT_URI=${APP_URL}/auth/callback
```

### Configuration in config/services.php

```php
'keycloak' => [
    'client_id' => env('KEYCLOAK_CLIENT_ID'),
    'client_secret' => env('KEYCLOAK_CLIENT_SECRET'),
    'redirect' => env('KEYCLOAK_REDIRECT_URI'),
    'base_url' => env('KEYCLOAK_BASE_URL'),
    'realm' => env('KEYCLOAK_REALM'),
],
```

---

## Socialite Keycloak Provider Setup

Install and configure the Keycloak Socialite provider:

```php
// config/app.php or AppServiceProvider boot()
// The socialiteproviders/keycloak package auto-registers via EventServiceProvider

// In app/Providers/EventServiceProvider.php
protected $listen = [
    \SocialiteProviders\Manager\SocialiteWasCalled::class => [
        \SocialiteProviders\Keycloak\KeycloakExtendSocialite::class . '@handle',
    ],
];
```

---

## Auth Controller

Complete controller handling OAuth2 login flow with Keycloak:

```php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\User;
use App\Services\KeycloakRoleMapper;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;
use Laravel\Socialite\Facades\Socialite;

class KeycloakAuthController extends Controller
{
    public function __construct(
        private readonly KeycloakRoleMapper $roleMapper,
    ) {}

    public function redirect(): RedirectResponse
    {
        return Socialite::driver('keycloak')
            ->scopes(['openid', 'profile', 'email'])
            ->redirect();
    }

    public function callback(): RedirectResponse
    {
        $keycloakUser = Socialite::driver('keycloak')->user();

        $user = User::updateOrCreate(
            ['keycloak_id' => $keycloakUser->getId()],
            [
                'name' => $keycloakUser->getName(),
                'email' => $keycloakUser->getEmail(),
                'username' => $keycloakUser->getNickname(),
            ]
        );

        // Sync roles from Keycloak token
        $roles = $this->roleMapper->extractRoles($keycloakUser->accessTokenResponseBody);
        $user->syncRoles($roles);

        Auth::login($user, remember: true);

        return redirect()->intended('/');
    }

    public function logout(): RedirectResponse
    {
        Auth::logout();
        request()->session()->invalidate();
        request()->session()->regenerateToken();

        // Redirect to Keycloak logout endpoint
        $keycloakLogoutUrl = config('services.keycloak.base_url')
            . '/realms/' . config('services.keycloak.realm')
            . '/protocol/openid-connect/logout'
            . '?redirect_uri=' . urlencode(config('app.url'));

        return redirect($keycloakLogoutUrl);
    }
}
```

---

## KeycloakRoleMapper

This class bridges Keycloak's OIDC token claim structure to spatie/laravel-permission
roles. It extracts roles from both `realm_access` and `resource_access` claims in the
token response.

```php
namespace App\Services;

use Illuminate\Support\Arr;

class KeycloakRoleMapper
{
    public function __construct(
        private readonly string $clientId,
    ) {}

    public static function make(): self
    {
        return new self(config('services.keycloak.client_id'));
    }

    /**
     * Extract roles from the Keycloak token response body.
     *
     * @param array $tokenResponse The full token response including 'id_token' claims
     * @return array<string> List of role names (e.g., ['HUB_ADMINISTRATOR', 'USER'])
     */
    public function extractRoles(array $tokenResponse): array
    {
        $roles = [];

        // Decode the ID token to get claims
        $idToken = $tokenResponse['id_token'] ?? null;
        if ($idToken) {
            $claims = json_decode(
                base64_decode(explode('.', $idToken)[1]),
                associative: true
            );

            // Extract realm roles
            $realmRoles = Arr::get($claims, 'realm_access.roles', []);
            $roles = array_merge($roles, $realmRoles);

            // Extract client-specific roles
            $clientRoles = Arr::get(
                $claims,
                "resource_access.{$this->clientId}.roles",
                []
            );
            $roles = array_merge($roles, $clientRoles);
        }

        // Normalize: uppercase, unique
        return array_unique(array_map('strtoupper', $roles));
    }
}
```

---

## Route Configuration

```php
// routes/web.php

use App\Http\Controllers\Auth\KeycloakAuthController;

// Public routes
Route::get('/auth/redirect', [KeycloakAuthController::class, 'redirect'])
    ->name('auth.redirect');
Route::get('/auth/callback', [KeycloakAuthController::class, 'callback'])
    ->name('auth.callback');

// Authenticated routes
Route::middleware(['auth', 'verified'])->group(function () {
    Route::post('/logout', [KeycloakAuthController::class, 'logout'])
        ->name('logout');

    // Module routes loaded from modules...
});
```

---

## Role-Based Access Control

### Using spatie/laravel-permission middleware

```php
// In module route files
Route::middleware(['role:HUB_ADMINISTRATOR'])->group(function () {
    Route::get('/corridor', [CorridorPageController::class, 'index']);
    Route::get('/corridor/{id}', [CorridorPageController::class, 'show']);
});

Route::middleware(['role:HUB_ADMINISTRATOR|HUB_OPERATION_SUPPORT'])->group(function () {
    Route::get('/employer', [EmployerPageController::class, 'index']);
});
```

### In Blade templates

```blade
@role('HUB_ADMINISTRATOR')
    <a href="/corridor">Corridor Management</a>
@endrole

@hasanyrole('HUB_ADMINISTRATOR|HUB_OPERATION_SUPPORT')
    <a href="/employer">Employers</a>
@endhasanyrole
```

### In controllers (Gate authorization)

```php
use Illuminate\Support\Facades\Gate;

public function destroy(string $id)
{
    Gate::authorize('delete-order');
    $this->orderService->delete($id);
    return redirect()->route('order.index');
}
```

### Role constants

Define roles as a constants class to avoid string duplication:

```php
namespace App\Constants;

final class Roles
{
    public const HUB_ADMINISTRATOR = 'HUB_ADMINISTRATOR';
    public const HUB_OPERATION_SUPPORT = 'HUB_OPERATION_SUPPORT';

    private function __construct() {}
}
```

---

## SecurityContext Helper

Utility to extract user information from the authenticated session. Wraps Laravel's
`auth()` helper for consistent access across services.

```php
namespace App\Services;

class SecurityContext
{
    public function getCurrentUserId(): ?string
    {
        return auth()->id();
    }

    public function getCurrentUsername(): ?string
    {
        return auth()->user()?->username;
    }

    public function getCurrentUserEmail(): ?string
    {
        return auth()->user()?->email;
    }

    public function getCurrentRoles(): array
    {
        $user = auth()->user();
        return $user ? $user->getRoleNames()->toArray() : [];
    }

    public function hasRole(string $role): bool
    {
        return auth()->user()?->hasRole($role) ?? false;
    }
}
```

This class is injected via constructor into:
- `BlameableTrait` — for `created_by` / `updated_by` fields
- Service classes — when business logic depends on the current user
- Correlation ID middleware — for logging

---

## BlameableTrait

Trait for Eloquent models to automatically populate `created_by` and `updated_by` fields.

```php
namespace App\Traits;

trait Blameable
{
    public static function bootBlameable(): void
    {
        static::creating(function ($model) {
            if (auth()->check()) {
                $model->created_by = auth()->user()->username ?? auth()->id();
                $model->updated_by = auth()->user()->username ?? auth()->id();
            } else {
                $model->created_by = 'system';
                $model->updated_by = 'system';
            }
        });

        static::updating(function ($model) {
            $model->updated_by = auth()->check()
                ? (auth()->user()->username ?? auth()->id())
                : 'system';
        });
    }
}
```

Usage in models:
```php
class Order extends Model
{
    use Blameable;

    // created_by and updated_by are auto-populated
}
```

---

## OIDC ID Token Structure Reference

For the spec, include a sample decoded ID token payload showing where Keycloak places
roles, so developers understand the claim structure:

```json
{
  "sub": "f1234-abcd-5678",
  "preferred_username": "john.doe",
  "email": "john.doe@example.com",
  "realm_access": {
    "roles": ["USER", "offline_access"]
  },
  "resource_access": {
    "{{KEYCLOAK_CLIENT_ID}}": {
      "roles": ["HUB_ADMINISTRATOR"]
    }
  },
  "scope": "openid profile email"
}
```

This helps developers understand exactly what `KeycloakRoleMapper` is parsing.

---

## CSP Nonce Middleware

Content Security Policy nonce generated per request to allow inline scripts while
maintaining CSP security.

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Vite;
use Illuminate\Support\Str;
use Symfony\Component\HttpFoundation\Response;

class CspNonceMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $nonce = Str::random(32);

        // Share nonce with Vite for script tags
        Vite::useCspNonce($nonce);

        // Store nonce for Blade templates
        view()->share('cspNonce', $nonce);

        $response = $next($request);

        $response->headers->set('Content-Security-Policy',
            "default-src 'self'; " .
            "script-src 'self' 'nonce-{$nonce}'; " .
            "style-src 'self' 'unsafe-inline';"
        );

        return $response;
    }
}
```

Register in `bootstrap/app.php`:
```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->web(append: [
        \App\Http\Middleware\CspNonceMiddleware::class,
    ]);
})
```

---

## ThemeMiddleware

Middleware to read the theme cookie and share with all views.

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class ThemeMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $theme = $request->cookie('theme', 'light');
        view()->share('theme', $theme);

        return $next($request);
    }
}
```

---

## User Model for Keycloak

```php
namespace App\Models;

use App\Traits\Blameable;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasRoles, Blameable;

    protected $fillable = [
        'keycloak_id',
        'name',
        'email',
        'username',
    ];

    protected $hidden = [
        'remember_token',
    ];
}
```
