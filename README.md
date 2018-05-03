# Laravel Passport Socialite
The missing social authentication plugin (i.e. SocialGrant) for laravel passport.

## Description
This package helps integrate social login using laravel's native packages i.e. (passport and socialite). This package allows social login from the providers that is supported 

## Getting Started
To get started add the following package to your composer.json file using this command.
`composer require anandsiddharth/laravel-paytm-wallet`

## Configuration
When composer installs this package successfully, register the   `Anand\Laravel\PassportSocialite\PassportSocialiteServiceProvider::class` in your config/app.php configuration file.


```
'providers' => [
    // Other service providers...
    Anand\Laravel\PassportSocialite\PassportSocialiteServiceProvider::class,
],
```

**Note: You need to configure third party social provider keys and secret strings as mentioned in laravel socialite documentation https://laravel.com/docs/5.6/socialite#configuration**

## Usage

### Step 1 - Setting up user model

Implement `UserSocialAccount` on your `User` model and then add method `findForPassportSocialite`.
`findForPassportSocialite` should accept two arguments i.e. `$provider` and `$id`
    
**$provider - string - will be the social provider i.e. facebook, google, github etc.**

**$id - string - is the user id as per social provider for example facebook's user id 1234567890**

**And the function should find the user which is related to that information and return user object or return null if not found**



Below is how your `User` model should look like after above implementations.

```php
namespace App;

use Anand\Laravel\PassportSocialite\User\UserSocialAccount;
class User extends Authenticatable implements UserSocialAccount {
    
    use HasApiTokens, Notifiable;

    /**
    * Find user from social provider's id
    * 
    * @param string $provider Provider name as requested from oauth e.g. facebook
    * @param string $id User id of social provider
    *
    * @return User
    */
    public static function findForPassportSocialite($provider,$id) {
        $account = SocialAccount::where('provider', $provider)->where('provider_user_id', $id)->first();
        if($account) {
            if($account->user){
                return $account->user;
            }
        }
        return;
    }
}
```

### Step 2 - Getting access token using social provider

I recommend you to not to request for access token from social grant directly from your app since the logic in social login is you need to create account if it doesn't exists or else login if account exists. 

So here in this case you will making a custom route and a controller that will recieve the Access Token or Authorization Token from your client i.e. Android, iOS etc. application.

Our route here can be something like this:

`Route::post('/auth/social/facebook', 'SocialLogin@loginFacebook');`

And here is how we can write our controller and its method for that :

```php
class SocialLogin extends Controller {

	public function loginFacebook(Request $request) {
		try {

			$facebook = Socialite::driver('facebook')->userFromToken($request->accessToken);
			if(!$exist = SocialAccount::where('provider',  SocialAccount::SERVICE_FACEBOOK)->where('provider_user_id', $facebook->getId())->first()){
				
				// create user account
			}
			return response()->json($this->issueToken('facebook', $request->accessToken));
		}
		catch(\Exception $e) {
			return response()->json([ "error" : $e->getMessage() ]);
		}
		
	}
    
	public function issueToken($provider, $accessToken) {
		$http = new GuzzleHttp\Client;

		$http->post('http://your-app.com/oauth/token', [
		    'form_params' => [
			'grant_type' => 'social',
			'client_id' => 'your-client-id', // it should be password grant client
			'client_secret' => 'client-secret',
			'accessToken' => $accessToken, // access token from provider
			'provider' => $provider,
		    ],
		]);
		return json_decode((string) $response->getBody(), true);
	}
}
```

**Note: SocialGrant will only accept access token not authorization token, for example google provides authorization token in android when requested server auth code i.e. offline access, so you need to exchange auth code for an access token. Refer here: https://github.com/google/google-api-php-client**

**Note: SocialGrant acts similar to PasswordGrant so make sure you use client id and secret of password grant while making oauth request**


**That's all folks**
