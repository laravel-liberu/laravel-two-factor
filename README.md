# Two-Factor-Laravel

[![Latest Version on Packagist](https://img.shields.io/packagist/v/emargareten/two-factor-laravel.svg?style=flat-square)](https://packagist.org/packages/emargareten/two-factor-laravel)
[![GitHub Tests Action Status](https://img.shields.io/github/actions/workflow/status/emargareten/two-factor-laravel/run-tests.yml?branch=master&label=tests&style=flat-square)](https://github.com/emargareten/two-factor-laravel/actions?query=workflow%3Arun-tests+branch%3Amaster)
[![GitHub Code Style Action Status](https://img.shields.io/github/actions/workflow/status/emargareten/two-factor-laravel/fix-php-code-style-issues.yml?branch=master&label=code%20style&style=flat-square)](https://github.com/emargareten/two-factor-laravel/actions?query=workflow%3A"Fix+PHP+code+style+issues"+branch%3Amaster)
[![Total Downloads](https://img.shields.io/packagist/dt/emargareten/two-factor-laravel.svg?style=flat-square)](https://packagist.org/packages/emargareten/two-factor-laravel)

Two-Factor-Laravel is a package that implements two-factor authentication for your Laravel apps.

## Installation

First, install the package into your project using composer:

```bash
composer require emargareten/two-factor-laravel
```

Next, you should publish the configuration and migration files using the `vendor:publish` Artisan command:

```bash
php artisan vendor:publish --provider="Emargareten\TwoFactor\ServiceProvider"
```

Finally, you should run your application's database migrations. This will add the two-factor columns to the `users` table:

```bash
php artisan migrate
```

### Configuration

After publishing the assets, you may review the `config/two-factor.php` configuration file. This file contains several options that allow you to customize the behavior of the two-factor authentication features.

## Usage

To start using two-factor authentication, you should first add the `TwoFactorAuthenticatable` trait to your `User` model:

```php
use Emargareten\TwoFactor\TwoFactorAuthenticatable;

class User extends Authenticatable
{
    use TwoFactorAuthenticatable;
}
```

### Enabling Two-Factor Authentication

This package provides the logic for authenticating users using two-factor authentication. However, it is up to you to provide the user interface and controllers for enabling and disabling two-factor authentication.

To enable two-factor authentication for a user, you should call the `EnableTwoFactorAuthentication` action, this will generate a secret key and recovery codes for the user and store them in the database (encrypted):

```php
use Emargareten\TwoFactor\Actions\DisableTwoFactorAuthentication;
use Emargareten\TwoFactor\Actions\EnableTwoFactorAuthentication;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class TwoFactorAuthenticationController extends Controller
{
    /**
     * Enable two-factor authentication for the user.
     */
    public function store(Request $request, EnableTwoFactorAuthentication $enable): RedirectResponse
    {
        $user = $request->user();

        if ($user->hasEnabledTwoFactorAuthentication()) {
            return back()->with('status', 'Two-factor authentication is already enabled');
        }

        if (! $user->hasPendingTwoFactorAuthentication()) {
            $enable($user);
        }

        return redirect()->route('account.two-factor-authentication.confirm.show');
    }
}
```

After enabling two-factor authentication, you should redirect the user to a page where they have to confirm enabling two-factor authentication by entering the one-time password generated by their authenticator app (or sent to them via SMS/email).

```php
use Emargareten\TwoFactor\Actions\ConfirmTwoFactorAuthentication;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\View\View;

class TwoFactorAuthenticationConfirmationController extends Controller
{
    /**
     * Get the two-factor authentication confirmation view.
     */
    public function show(Request $request): View|RedirectResponse
    {
        $user = $request->user();

        if ($user->hasEnabledTwoFactorAuthentication()) {
            return back()->with('status', 'Two-factor authentication is already enabled');
        }

        if (! $user->hasPendingTwoFactorAuthentication()) {
            return back()->with('status', 'Two-factor authentication is disabled');
        }

        return view('account.two-factor-confirmation.show', [
            'qrCodeSvg' => $user->twoFactorQrCodeSvg(),
            'setupKey' => $request->user()->two_factor_secret,
        ]);
    }

    /**
     * Confirm two-factor authentication for the user.
     */
    public function store(Request $request, ConfirmTwoFactorAuthentication $confirm): RedirectResponse
    {
        $request->validate([
            'code' => ['required', 'string'],
        ]);

        $confirm($request->user(), $request->code);

        return redirect()
            ->route('account.two-factor-authentication.recovery-codes.index')
            ->with('status', 'Two-factor authentication successfully enabled');
    }
}
```

If you prefer to use a different method for receiving the one-time password, i.e. SMS/email, you can use the `getCurrentOtp` method on the user model to get retrieve current one-time password:

```php
$user->getCurrentOtp();
```

> **Note**
> When sending the one-time-password via SMS/email, you should set the window config to a higher value, i.e. 10, to allow the user to enter the one-time password after it has been sent.

You should also provide a way for the user to disable two-factor authentication. This can be done by calling the `DisableTwoFactorAuthentication` action:

```php
/**
 * Disable two-factor authentication for the user.
 */
public function destroy(Request $request, DisableTwoFactorAuthentication $disable): RedirectResponse
{
    $disable($request->user());

    return back()->with('status', 'Two-factor authentication disabled successfully');
}
```

Once the user has confirmed enabling two-factor authentication, each time they log in, they will be redirected to a page where they will be asked to enter a one-time password generated by their authenticator app.

```php
use Emargareten\TwoFactor\Actions\TwoFactorRedirector;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

public function login(Request $request, TwoFactorRedirector $redirector): Response
{
    // do login stuff...

    return $redirector->redirect($request);
}
```

or if the user should retrieve the OTP via other methods:

```php
 $redirect = $redirector->redirect($request);
 
 if ($redirector->isRedirectingToChallenge()) {
     $request->user()->notify(new SendOTP);
 }
 
 return $redirect;
```

This will redirect the user to the `two-factor-challenge.create` route.

You will need to provide a view for this route. This view should contain a form where the user can enter the one-time password, you should bind the view to the `TwoFactorChallengeViewResponse::class` in the `register` method of your `AppServiceProvider` by calling the `TwoFactor::challengeView()` method:

```php
/**
 * Register any application services.
 */
public function register(): void
{
    TwoFactor::challengeView('two-factor-challenge.create');
}
```

Or use a closure to generate a custom response:

```php
TwoFactor::challengeView(function (Request $request)  {
    return Inertia::render('TwoFactorChallenge/Create');
});
```

The form should be submitted to the `two-factor-challenge.store` route.

Once the user has entered a valid one-time password, he will be redirected to the intended URL (or to the home route defined in the config file if no intended URL was set).

### Recovery Codes

This package also provides the logic for generating and using recovery codes. Recovery codes can be used to access the application in case the user loses access to their authenticator app.

After enabling two-factor authentication, you should redirect the user to a page where they can view their recovery codes. You can generate a fresh set of recovery codes by calling the `GenerateNewRecoveryCodes` action:

```php
use Emargareten\TwoFactor\Actions\GenerateNewRecoveryCodes;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\View\View;

class TwoFactorAuthenticationRecoveryCodeController extends Controller
{
    /**
     * Get the two-factor authentication recovery codes for authenticated user.
     */
    public function index(Request $request): View|RedirectResponse
    {
        if (! $request->user()->hasEnabledTwoFactorAuthentication()) {
            return back()->with('status', 'Two-factor authentication is disabled');
        }

        return view('two-factor-recovery-codes.index', [
            'recoveryCodes' => $request->user()->two_factor_recovery_codes,
        ]);
    }

    /**
     * Generate a fresh set of two-factor authentication recovery codes.
     */
    public function store(Request $request, GenerateNewRecoveryCodes $generate): RedirectResponse
    {
        if (! $request->user()->hasEnabledTwoFactorAuthentication()) {
            return back()->with('status', 'Two-factor authentication is disabled');
        }

        $generate($request->user());

        return redirect()->route('account.two-factor-authentication.recovery-codes.index');
    }
}
```

To use the recovery codes, you should add a view for the `two-factor-challenge-recovery.create` route. This view should contain a form where the user can enter a recovery code. You should bind the view to the `TwoFactorChallengeRecoveryViewResponse::class` in the `register` method of your `AppServiceProvider` by calling the `TwoFactor::challengeRecoveryView()` method:

The form should be submitted to the `two-factor-challenge-recovery.store` route.

## Testing

```bash
composer test
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Security Vulnerabilities

Please review [our security policy](../../security/policy) on how to report security vulnerabilities.

## Credits

- [emargareten](https://github.com/emargareten)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
