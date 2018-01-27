# Laravel Passport——OAuth2 API 认证系统源码解析（上）

## 前言

在 Laravel 中，实现基于传统表单的登陆和授权已经非常简单，但是如何满足 API 场景下的授权需求呢？在 API 场景里通常通过令牌来实现用户授权，而非维护请求之间的 Session 状态。在 Laravel 项目中使用 Passport 可以轻而易举地实现 API 授权认证，Passport 可以在几分钟之内为你的应用程序提供完整的 OAuth2 服务端实现。

首先我们可以先了解一下 OAuth2 : [理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

可以看出来，OAuth2 的授权模式分为 4 种，相应的 Passport 的授权模式也是 4 中。下面，我们就会逐一进行源码分析。

## Passport 服务的注册启动

```php
class PassportServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->registerAuthorizationServer();

        $this->registerResourceServer();

        $this->registerGuard();
    }
}

```

我们知道 `OAuth2` 大致由 `客户`、`客户端`、`认证服务器`、`资源服务器` 等构成。 在这里，我们扮演着 `认证服务器` 与 `资源服务器` 的角色。

### 认证服务器注册

```php
protected function registerAuthorizationServer()
{
    $this->app->singleton(AuthorizationServer::class, function () {
        return tap($this->makeAuthorizationServer(), function ($server) {
            $server->enableGrantType(
                $this->makeAuthCodeGrant(), Passport::tokensExpireIn()
            );

            $server->enableGrantType(
                $this->makeRefreshTokenGrant(), Passport::tokensExpireIn()
            );

            $server->enableGrantType(
                $this->makePasswordGrant(), Passport::tokensExpireIn()
            );

            $server->enableGrantType(
                new PersonalAccessGrant, new DateInterval('P1Y')
            );

            $server->enableGrantType(
                new ClientCredentialsGrant, Passport::tokensExpireIn()
            );

            if (Passport::$implicitGrantEnabled) {
                $server->enableGrantType(
                    $this->makeImplicitGrant(), Passport::tokensExpireIn()
                );
            }
        });
    });
}

```

`AuthorizationServer` 认证服务器是 ` League OAuth2 server` 的一个类，是 `League` 关于 `OAuth2` 的实现类。这个认证服务器类需要 5 个参数，分别代表 `客户端`、`token 令牌`、`scope 作用范围`、`加密私钥`、`加密 key`。

```php
class AuthorizationServer implements EmitterAwareInterface
{
    public function __construct(
        ClientRepositoryInterface $clientRepository,
        AccessTokenRepositoryInterface $accessTokenRepository,
        ScopeRepositoryInterface $scopeRepository,
        $privateKey,
        $encryptionKey,
        ResponseTypeInterface $responseType = null
    ) {
        $this->clientRepository = $clientRepository;
        $this->accessTokenRepository = $accessTokenRepository;
        $this->scopeRepository = $scopeRepository;

        if ($privateKey instanceof CryptKey === false) {
            $privateKey = new CryptKey($privateKey);
        }
        $this->privateKey = $privateKey;
        $this->encryptionKey = $encryptionKey;
        $this->responseType = $responseType;
    }
}
```

这些不同的 `Repository` 均是各个接口类，这些类规定了各个部分的功能。`Passport` 实现了上述几个接口类：

```php
public function makeAuthorizationServer()
{
    return new AuthorizationServer(
        $this->app->make(Bridge\ClientRepository::class),
        $this->app->make(Bridge\AccessTokenRepository::class),
        $this->app->make(Bridge\ScopeRepository::class),
        $this->makeCryptKey('oauth-private.key'),
        app('encrypter')->getKey()
    );
}

protected function makeCryptKey($key)
{
    return new CryptKey(
        'file://'.Passport::keyPath($key),
        null,
        false
    );
}
```

`oauth-private.key` 这个私钥由 `php artisan passport:keys` 命令生成。`encrypter` 的加密 `key` 是 `.env` 文件的 `key` 属性。

构建认证服务器之后，还要对认证服务器注册授权方式。 `Passport` 的授权方式有传统的 `OAuth2` : `授权码模式`、`密码模式`、`隐性模式`、`客户端模式`，还有 `刷新令牌模式`、`个人授权模式` 等。

```php
protected function makeAuthCodeGrant()
{
    return tap($this->buildAuthCodeGrant(), function ($grant) {
        $grant->setRefreshTokenTTL(Passport::refreshTokensExpireIn());
    });
}

protected function buildAuthCodeGrant()
{
    return new AuthCodeGrant(
        $this->app->make(Bridge\AuthCodeRepository::class),
        $this->app->make(Bridge\RefreshTokenRepository::class),
        new DateInterval('PT10M')
    );
}
```

### 资源服务器注册

类似的, `ResourceServer` 也是 `League` 的资源服务器类：

```php
protected function registerResourceServer()
{
    $this->app->singleton(ResourceServer::class, function () {
        return new ResourceServer(
            $this->app->make(Bridge\AccessTokenRepository::class),
            $this->makeCryptKey('oauth-public.key')
        );
    });
}
```

### guard 注册

当我们已经构建好 `Passport` 服务之后，我们只要利用中间件 `Auth:api` 就可以利用 `Passport` 验证 `api` 的合法性。具体的原理是 中间件 `Auth` 的参数 `api` 是指定 `guard` 的名称，例如 `web`、`api`,如果调用的是 `api` 的 `guard` 那么就会创建相应的 `passport` 驱动器：
 
```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],

```

而 `passport` 的 `guard` 驱动器就是这个 `TokenGuard`:	

```php
protected function registerGuard()
{
    Auth::extend('passport', function ($app, $name, array $config) {
        return tap($this->makeGuard($config), function ($guard) {
            $this->app->refresh('request', $guard, 'setRequest');
        });
    });
}
    
protected function makeGuard(array $config)
{
    return new RequestGuard(function ($request) use ($config) {
        return (new TokenGuard(
            $this->app->make(ResourceServer::class),
            Auth::createUserProvider($config['provider']),
            $this->app->make(TokenRepository::class),
            $this->app->make(ClientRepository::class),
            $this->app->make('encrypter')
        ))->user($request);
    }, $this->app['request']);
}
```

## 授权码模式

授权码模式大概分为 5 个步骤：

- 第三方 向我们的服务器申请创建客户端。
- 用户打开客户端以后，客户端会跳转到我们的网站授权页面要求用户给予授权。
- 用户同意给予客户端授权，我们将会返回 `授权码`。
- 客户端使用上一步获得的授权，向认证服务器申请令牌。
- 客户端使用令牌，向资源服务器申请获取资源。

为何授权码模式需要如此设置步骤可以查看：[Why is there an “Authorization Code” flow in OAuth2 when “Implicit” flow works so well?](https://stackoverflow.com/questions/13387698/why-is-there-an-authorization-code-flow-in-oauth2-when-implicit-flow-works-s)、[OAuth2疑问解答](http://bylijinnan.iteye.com/blog/2277548)

### 创建客户端

在创建客户端这一步骤，第三方需要提供客户端名称与客户端的 `redirect` ：

```php
const data = {
    name: 'Client Name',
    redirect: 'http://example.com/callback'
};

axios.post('/oauth/clients', data)
    .then(response => {
        console.log(response.data);
    })
    .catch (response => {
        // List errors on response...
    });

```
我们在创建成功之后，会返回此客户端的 `ID` 和密钥。这两个东西十分重要，是后面几个步骤必要的参数。

```php
public function forClients()
{
    $this->router->group(['middleware' => ['web', 'auth']], function ($router) {
        $router->get('/clients', [
            'uses' => 'ClientController@forUser',
        ]);

        $router->post('/clients', [
            'uses' => 'ClientController@store',
        ]);

        $router->put('/clients/{client_id}', [
            'uses' => 'ClientController@update',
        ]);

        $router->delete('/clients/{client_id}', [
            'uses' => 'ClientController@destroy',
        ]);
    });
}

public function store(Request $request)
{
    $this->validation->make($request->all(), [
        'name' => 'required|max:255',
        'redirect' => 'required|url',
    ])->validate();

    return $this->clients->create(
        $request->user()->getKey(), $request->name, $request->redirect
    )->makeVisible('secret');
}

public function create($userId, $name, $redirect, $personalAccess = false, $password = false)
{
    $client = (new Client)->forceFill([
        'user_id' => $userId,
        'name' => $name,
        'secret' => str_random(40),
        'redirect' => $redirect,
        'personal_access_client' => $personalAccess,
        'password_client' => $password,
        'revoked' => false,
    ]);

    $client->save();

    return $client;
}
```

### 跳转授权页面

客户端创建之后，开发者会使用此客户端的 ID 和密钥来请求授权代码，并从应用程序访问令牌。首先，接入应用的用户向你应用程序的 /oauth/authorize 路由发出重定向请求，

```php
Route::get('/redirect', function () {
    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://example.com/callback',
        'response_type' => 'code',
        'scope' => '',
    ]);

    return redirect('http://your-app.com/oauth/authorize?'.$query);
});

```
这个链接会访问我们的授权路由，我们的服务器会验证上面的四个参数，考察是否存在这个第三方客户端，如果验证通过，将会渲染出我们的授权页面。

```php
public function forAuthorization()
{
    $this->router->group(['middleware' => ['web', 'auth']], function ($router) {
        $router->get('/authorize', [
            'uses' => 'AuthorizationController@authorize',
        ]);

        $router->post('/authorize', [
            'uses' => 'ApproveAuthorizationController@approve',
        ]);

        $router->delete('/authorize', [
            'uses' => 'DenyAuthorizationController@deny',
        ]);
    });
}

public function authorize(ServerRequestInterface $psrRequest,
                          Request $request,
                          ClientRepository $clients,
                          TokenRepository $tokens)
{
    return $this->withErrorHandling(function () use ($psrRequest, $request, $clients, $tokens) {
        $authRequest = $this->server->validateAuthorizationRequest($psrRequest);

        $scopes = $this->parseScopes($authRequest);

        $token = $tokens->findValidToken(
            $user = $request->user(),
            $client = $clients->find($authRequest->getClient()->getIdentifier())
        );

        if ($token && $token->scopes === collect($scopes)->pluck('id')->all()) {
            return $this->approveRequest($authRequest, $user);
        }

        $request->session()->put('authRequest', $authRequest);

        return $this->response->view('passport::authorize', [
            'client' => $client,
            'user' => $user,
            'scopes' => $scopes,
            'request' => $request,
        ]);
    });
}

```

这里最关键的就是 `validateAuthorizationRequest` 这个函数：

```php
public function validateAuthorizationRequest(ServerRequestInterface $request)
{
    foreach ($this->enabledGrantTypes as $grantType) {
        if ($grantType->canRespondToAuthorizationRequest($request)) {
            return $grantType->validateAuthorizationRequest($request);
        }
    }

    throw OAuthServerException::unsupportedGrantType();
}
```

`canRespondToAuthorizationRequest` 用于验证授权模式与参数的 `response_type` 是否符合。如果确认授权模式正确，那么接下来就会继续验证以下几项：

- 客户端 id
- redirect 重定向地址
- scopes 授权范围
- state 客户端状态

#### 客户端

```php
public function validateAuthorizationRequest(ServerRequestInterface $request)
{
    $clientId = $this->getQueryStringParameter(
        'client_id',
        $request,
        $this->getServerParameter('PHP_AUTH_USER', $request)
    );
    if (is_null($clientId)) {
        throw OAuthServerException::invalidRequest('client_id');
    }

    $client = $this->clientRepository->getClientEntity(
        $clientId,
        $this->getIdentifier(),
        null,
        false
    );

    if ($client instanceof ClientEntityInterface === false) {
        $this->getEmitter()->emit(new RequestEvent(RequestEvent::CLIENT_AUTHENTICATION_FAILED, $request));
        throw OAuthServerException::invalidClient();
    }

    ...
}

public function getIdentifier()
{
    return 'authorization_code';
}
```
客户端的验证主要是利用请求中的参数 `client_id`，我们会从表 `oauth_clients` 的表中按照 `client_id` 来取出数据库记录：

```php
public function getClientEntity($clientIdentifier, $grantType,
                                    $clientSecret = null, $mustValidateSecret = true)
{
    $record = $this->clients->findActive($clientIdentifier);

    if (! $record || ! $this->handlesGrant($record, $grantType)) {
        return;
    }

    $client = new Client(
        $clientIdentifier, $record->name, $record->redirect
    );

    if ($mustValidateSecret &&
        ! hash_equals($record->secret, (string) $clientSecret)) {
        return;
    }

    return $client;
}

public function findActive($id)
{
    $client = $this->find($id);

    return $client && ! $client->revoked ? $client : null;
}

protected function handlesGrant($record, $grantType)
{
    switch ($grantType) {
        case 'authorization_code':
            return ! $record->firstParty();
        case 'personal_access':
            return $record->personal_access_client;
        case 'password':
            return $record->password_client;
        default:
            return true;
    }
}
```

在表 `oauth_clients` 中还有两个字段 `personal_access`、`password`，对于授权码模式来说这两个字段都要求为 0。

#### 重定向地址 

```php
public function validateAuthorizationRequest(ServerRequestInterface $request)
{
    ...
    
    $redirectUri = $this->getQueryStringParameter('redirect_uri', $request);
    if ($redirectUri !== null) {
        if (
            is_string($client->getRedirectUri())
            && (strcmp($client->getRedirectUri(), $redirectUri) !== 0)
        ) {
            $this->getEmitter()->emit(new RequestEvent(RequestEvent::CLIENT_AUTHENTICATION_FAILED, $request));
            throw OAuthServerException::invalidClient();
        } elseif (
            is_array($client->getRedirectUri())
            && in_array($redirectUri, $client->getRedirectUri()) === false
        ) {
            $this->getEmitter()->emit(new RequestEvent(RequestEvent::CLIENT_AUTHENTICATION_FAILED, $request));
            throw OAuthServerException::invalidClient();
        }
    } elseif (is_array($client->getRedirectUri()) && count($client->getRedirectUri()) !== 1
        || empty($client->getRedirectUri())
    ) {
        $this->getEmitter()->emit(new RequestEvent(RequestEvent::CLIENT_AUTHENTICATION_FAILED, $request));
        throw OAuthServerException::invalidClient();
    }
    
    ...

}
```

这部分验证参数中的 `redirect_uri` 是否与数据库中的重定向地址是否一致。


#### 授权作用域

授权作用域可以让 API 客户端在请求账户授权时请求特定的权限。例如，如果你正在构建电子商务应用程序，并不是所有接入的 API 应用都需要下订单的功能。你可以让接入的 API 应用只被允许授权访问订单发货状态。换句话说，作用域允许应用程序的用户限制第三方应用程序执行的操作。

你可以在 AuthServiceProvider 的 boot 方法中使用 Passport::tokensCan 方法来定义 API 的作用域。tokensCan 方法接受一个作用域名称、描述的数组作为参数。作用域描述将会在授权确认页中直接展示给用户，你可以将其定义为任何你需要的内容：

```php
Passport::tokensCan([
    'place-orders' => 'Place orders',
    'check-status' => 'Check order status',
]);

public static function tokensCan(array $scopes)
{
    static::$scopes = $scopes;
}
```

验证授权作用域的时候，只是在 `Passport` 中验证是否存在该授权作用域：

```php
public function validateAuthorizationRequest(ServerRequestInterface $request)
{
    ...
    
    $scopes = $this->validateScopes(
        $this->getQueryStringParameter('scope', $request, $this->defaultScope),
        is_array($client->getRedirectUri())
            ? $client->getRedirectUri()[0]
            : $client->getRedirectUri()
    );
    
    ...
}

public function validateScopes($scopes, $redirectUri = null)
{
    $scopesList = array_filter(explode(self::SCOPE_DELIMITER_STRING, trim($scopes)), function ($scope) {
        return !empty($scope);
    });

    $validScopes = [];

    foreach ($scopesList as $scopeItem) {
        $scope = $this->scopeRepository->getScopeEntityByIdentifier($scopeItem);

        if ($scope instanceof ScopeEntityInterface === false) {
            throw OAuthServerException::invalidScope($scopeItem, $redirectUri);
        }

        $validScopes[] = $scope;
    }

    return $validScopes;
}

public function getScopeEntityByIdentifier($identifier)
{
    if (Passport::hasScope($identifier)) {
        return new Scope($identifier);
    }
}
```

#### state 

这个字段用于防止 csrf 攻击的，具体可以查看 ：[移花接木：针对OAuth2的CSRF攻击](https://www.jianshu.com/p/c7c8f51713b6)

```php
public function validateAuthorizationRequest(ServerRequestInterface $request)
{
    $stateParameter = $this->getQueryStringParameter('state', $request);

        $authorizationRequest = new AuthorizationRequest();
        $authorizationRequest->setGrantTypeId($this->getIdentifier());
        $authorizationRequest->setClient($client);
        $authorizationRequest->setRedirectUri($redirectUri);
        $authorizationRequest->setState($stateParameter);
        $authorizationRequest->setScopes($scopes);
}
```

验证结束后，接下来就会验证当前用户是否已经授权过，如果已经授权过，那么就会直接返回授权码，否则就会渲染授权页面：

```php
public function authorize(ServerRequestInterface $psrRequest,
                              Request $request,
                              ClientRepository $clients,
                              TokenRepository $tokens)
{
    return $this->withErrorHandling(function () use ($psrRequest, $request, $clients, $tokens) {
        $authRequest = $this->server->validateAuthorizationRequest($psrRequest);

        $scopes = $this->parseScopes($authRequest);

        $token = $tokens->findValidToken(
            $user = $request->user(),
            $client = $clients->find($authRequest->getClient()->getIdentifier())
        );

        if ($token && $token->scopes === collect($scopes)->pluck('id')->all()) {
            return $this->approveRequest($authRequest, $user);
        }

        $request->session()->put('authRequest', $authRequest);

        return $this->response->view('passport::authorize', [
            'client' => $client,
            'user' => $user,
            'scopes' => $scopes,
            'request' => $request,
        ]);
    });
}
```

验证用户的是否授权首先是查看授权作用域是否与数据库保持一致。由于授权作用域与 `token` 相互关联，并非与客户端相互关联，所以 `scopes` 没有在 `oauth_clients` 表中，而是在 `oauth_access_tokens` 这个表中。

```php
protected function parseScopes($authRequest)
{
    return Passport::scopesFor(
        collect($authRequest->getScopes())->map(function ($scope) {
            return $scope->getIdentifier();
        })->all()
    );
}

public static function scopesFor(array $ids)
{
    return collect($ids)->map(function ($id) {
        if (isset(static::$scopes[$id])) {
            return new Scope($id, static::$scopes[$id]);
        }

        return;
    })->filter()->values()->all();
}
```

可以看到，作用域的 `identifier` 就是 `Scope` 的 `id`。

#### 获取已授权 token

token 的获取主要是利用 `client_id` 与 `user_id` 在表 `oauth_access_tokens` 中查询符合条件的 `token`。

```php
public function findValidToken($user, $client)
{
    return $client->tokens()
                  ->whereUserId($user->getKey())
                  ->whereRevoked(0)
                  ->where('expires_at', '>', Carbon::now())
                  ->latest('expires_at')
                  ->first();
}

public function tokens()
{
    return $this->hasMany(Token::class, 'client_id');
}
```

在获取到有效的 `token` 之后，并且 `token` 的作用域符合请求参数，就会立即返回，不需要用户的重复授权：

```php
protected function approveRequest($authRequest, $user)
{
    $authRequest->setUser(new User($user->getKey()));

    $authRequest->setAuthorizationApproved(true);

    return $this->convertResponse(
        $this->server->completeAuthorizationRequest($authRequest, new Psr7Response)
    );
}
```

### 授权成功

用户点击确认按钮，授权成功之后，服务器就会跳转到客户端预设的 `redirecturi`，并且携带授权码等一系列参数

```php
$router->post('/authorize', [
    'uses' => 'ApproveAuthorizationController@approve',
]);


public function approve(Request $request)
{
    return $this->withErrorHandling(function () use ($request) {
        $authRequest = $this->getAuthRequestFromSession($request);

        return $this->convertResponse(
            $this->server->completeAuthorizationRequest($authRequest, new Psr7Response)
        );
    });
}
```

`completeAuthorizationRequest` 是授权服务器的重要步骤：

```php
public function completeAuthorizationRequest(AuthorizationRequest $authRequest, ResponseInterface $response)
{
    return $this->enabledGrantTypes[$authRequest->getGrantTypeId()]
        ->completeAuthorizationRequest($authRequest)
        ->generateHttpResponse($response);
}


public function completeAuthorizationRequest(AuthorizationRequest $authorizationRequest)
{
    if ($authorizationRequest->getUser() instanceof UserEntityInterface === false) {
        throw new \LogicException('An instance of UserEntityInterface should be set on the AuthorizationRequest');
    }

    $finalRedirectUri = ($authorizationRequest->getRedirectUri() === null)
        ? is_array($authorizationRequest->getClient()->getRedirectUri())
            ? $authorizationRequest->getClient()->getRedirectUri()[0]
            : $authorizationRequest->getClient()->getRedirectUri()
        : $authorizationRequest->getRedirectUri();

    // The user approved the client, redirect them back with an auth code
    if ($authorizationRequest->isAuthorizationApproved() === true) {
        $authCode = $this->issueAuthCode(
            $this->authCodeTTL,
            $authorizationRequest->getClient(),
            $authorizationRequest->getUser()->getIdentifier(),
            $authorizationRequest->getRedirectUri(),
            $authorizationRequest->getScopes()
        );

        $payload = [
            'client_id'             => $authCode->getClient()->getIdentifier(),
            'redirect_uri'          => $authCode->getRedirectUri(),
            'auth_code_id'          => $authCode->getIdentifier(),
            'scopes'                => $authCode->getScopes(),
            'user_id'               => $authCode->getUserIdentifier(),
            'expire_time'           => (new \DateTime())->add($this->authCodeTTL)->format('U'),
            'code_challenge'        => $authorizationRequest->getCodeChallenge(),
            'code_challenge_method' => $authorizationRequest->getCodeChallengeMethod(),
        ];

        $response = new RedirectResponse();
        $response->setRedirectUri(
            $this->makeRedirectUri(
                $finalRedirectUri,
                [
                    'code'  => $this->encrypt(
                        json_encode(
                            $payload
                        )
                    ),
                    'state' => $authorizationRequest->getState(),
                ]
            )
        );

        return $response;
    }

    // The user denied the client, redirect them back with an error
    throw OAuthServerException::accessDenied(
        'The user denied the request',
        $this->makeRedirectUri(
            $finalRedirectUri,
            [
                'state' => $authorizationRequest->getState(),
            ]
        )
    );
}
```

这里最重要的就是 `issueAuthCode` 生成授权码：


```php
protected function issueAuthCode(
    \DateInterval $authCodeTTL,
    ClientEntityInterface $client,
    $userIdentifier,
    $redirectUri,
    array $scopes = []
) {
    $maxGenerationAttempts = self::MAX_RANDOM_TOKEN_GENERATION_ATTEMPTS;

    $authCode = $this->authCodeRepository->getNewAuthCode();
    $authCode->setExpiryDateTime((new \DateTime())->add($authCodeTTL));
    $authCode->setClient($client);
    $authCode->setUserIdentifier($userIdentifier);
    $authCode->setRedirectUri($redirectUri);

    foreach ($scopes as $scope) {
        $authCode->addScope($scope);
    }

    while ($maxGenerationAttempts-- > 0) {
        $authCode->setIdentifier($this->generateUniqueIdentifier());
        try {
            $this->authCodeRepository->persistNewAuthCode($authCode);

            return $authCode;
        } catch (UniqueTokenIdentifierConstraintViolationException $e) {
            if ($maxGenerationAttempts === 0) {
                throw $e;
            }
        }
    }
}
```

其中 `generateUniqueIdentifier` 就是生成授权码的步骤，这个授权码也是表 `oauth_auth_codes` 的 `id`：

```php
protected function generateUniqueIdentifier($length = 40)
{
    try {
        return bin2hex(random_bytes($length));
        // @codeCoverageIgnoreStart
    } catch (\TypeError $e) {
        throw OAuthServerException::serverError('An unexpected error has occurred');
    } catch (\Error $e) {
        throw OAuthServerException::serverError('An unexpected error has occurred');
    } catch (\Exception $e) {
        // If you get this message, the CSPRNG failed hard.
        throw OAuthServerException::serverError('Could not generate a random string');
    }
    // @codeCoverageIgnoreEnd
}

public function persistNewAuthCode(AuthCodeEntityInterface $authCodeEntity)
{
    $this->database->table('oauth_auth_codes')->insert([
        'id' => $authCodeEntity->getIdentifier(),
        'user_id' => $authCodeEntity->getUserIdentifier(),
        'client_id' => $authCodeEntity->getClient()->getIdentifier(),
        'scopes' => $this->formatScopesForStorage($authCodeEntity->getScopes()),
        'revoked' => false,
        'expires_at' => $authCodeEntity->getExpiryDateTime(),
    ]);
}
```

### 授权码转为令牌

由于 `client_id` 是公开的，因此上一步授权码的获取理论上很容易，真正重要的是授权码转为令牌：

```php
Route::get('/callback', function (Request $request) {
    $http = new GuzzleHttp\Client;

    $response = $http->post('http://your-app.com/oauth/token', [
        'form_params' => [
            'grant_type' => 'authorization_code',
            'client_id' => 'client-id',
            'client_secret' => 'client-secret',
            'redirect_uri' => 'http://example.com/callback',
            'code' => $request->code,
        ],
    ]);

    return json_decode((string) $response->getBody(), true);
});
```

这一步需要客户端提供注册时返回的密码，

```php
public function forAccessTokens()
{
    $this->router->post('/token', [
        'uses' => 'AccessTokenController@issueToken',
        'middleware' => 'throttle',
    ]);
}

public function issueToken(ServerRequestInterface $request)
{
    return $this->withErrorHandling(function () use ($request) {
        return $this->convertResponse(
            $this->server->respondToAccessTokenRequest($request, new Psr7Response)
        );
    });
}
```

这一步需要验证的东西非常繁多，我们分部分来看：

#### 客户端验证

客户端验证主要校验 `client_id`、 `client_secret`、`redirect_uri` :

```php
public function respondToAccessTokenRequest(
    ServerRequestInterface $request,
    ResponseTypeInterface $responseType,
    \DateInterval $accessTokenTTL
) {
    
    $client = $this->validateClient($request);
    
    ...
    
}

protected function validateClient(ServerRequestInterface $request)
{
    list($basicAuthUser, $basicAuthPassword) = $this->getBasicAuthCredentials($request);

    $clientId = $this->getRequestParameter('client_id', $request, $basicAuthUser);

    // If the client is confidential require the client secret
    $clientSecret = $this->getRequestParameter('client_secret', $request, $basicAuthPassword);

    $client = $this->clientRepository->getClientEntity(
        $clientId,
        $this->getIdentifier(),
        $clientSecret,
        true
    );
    
    $redirectUri = $this->getRequestParameter('redirect_uri', $request, null);

    return $client;
}

protected function getBasicAuthCredentials(ServerRequestInterface $request)
{
    if (!$request->hasHeader('Authorization')) {
        return [null, null];
    }

    $header = $request->getHeader('Authorization')[0];
    if (strpos($header, 'Basic ') !== 0) {
        return [null, null];
    }

    if (!($decoded = base64_decode(substr($header, 6)))) {
        return [null, null];
    }

    if (strpos($decoded, ':') === false) {
        return [null, null]; // HTTP Basic header without colon isn't valid
    }

    return explode(':', $decoded, 2);
}

```

#### 验证授权码

客户端的密码验证通过后，就会开始验证授权码，授权码的验证主要涉及 `expire_time`、`client_id`、`auth_code_id`:

```php
public function respondToAccessTokenRequest(
    ServerRequestInterface $request,
    ResponseTypeInterface $responseType,
    \DateInterval $accessTokenTTL
) {

    ...
    
    $encryptedAuthCode = $this->getRequestParameter('code', $request, null);

    if ($encryptedAuthCode === null) {
        throw OAuthServerException::invalidRequest('code');
    }

    try {
        $authCodePayload = json_decode($this->decrypt($encryptedAuthCode));
        if (time() > $authCodePayload->expire_time) {
            throw OAuthServerException::invalidRequest('code', 'Authorization code has expired');
        }

        if ($this->authCodeRepository->isAuthCodeRevoked($authCodePayload->auth_code_id) === true) {
            throw OAuthServerException::invalidRequest('code', 'Authorization code has been revoked');
        }

        if ($authCodePayload->client_id !== $client->getIdentifier()) {
            throw OAuthServerException::invalidRequest('code', 'Authorization code was not issued to this client');
        }

        // The redirect URI is required in this request
        $redirectUri = $this->getRequestParameter('redirect_uri', $request, null);
        if (empty($authCodePayload->redirect_uri) === false && $redirectUri === null) {
            throw OAuthServerException::invalidRequest('redirect_uri');
        }

        if ($authCodePayload->redirect_uri !== $redirectUri) {
            throw OAuthServerException::invalidRequest('redirect_uri', 'Invalid redirect URI');
        }

    } catch (\LogicException  $e) {
        throw OAuthServerException::invalidRequest('code', 'Cannot decrypt the authorization code');
    }
}

public function isAuthCodeRevoked($codeId)
{
    return $this->database->table('oauth_auth_codes')
                ->where('id', $codeId)->where('revoked', 1)->exists();
}
```

#### 发放令牌

令牌的发放主要是 `access_token`、`refresh_token`，并且取消相关的授权码：

```php
public function respondToAccessTokenRequest(
    ServerRequestInterface $request,
    ResponseTypeInterface $responseType,
    \DateInterval $accessTokenTTL
) {

    // Issue and persist access + refresh tokens
    $accessToken = $this->issueAccessToken($accessTokenTTL, $client, $authCodePayload->user_id, $scopes);
    $refreshToken = $this->issueRefreshToken($accessToken);

    // Inject tokens into response type
    $responseType->setAccessToken($accessToken);
    $responseType->setRefreshToken($refreshToken);

    // Revoke used auth code
    $this->authCodeRepository->revokeAuthCode($authCodePayload->auth_code_id);

    return $responseType;
}
```
首先需要生成 `access_token`，之后再对表 `oauth_access_tokens` 持久化 `access_token`：

```php
protected function issueAccessToken(
    \DateInterval $accessTokenTTL,
    ClientEntityInterface $client,
    $userIdentifier,
    array $scopes = []
) {
    $maxGenerationAttempts = self::MAX_RANDOM_TOKEN_GENERATION_ATTEMPTS;

    $accessToken = $this->accessTokenRepository->getNewToken($client, $scopes, $userIdentifier);
    $accessToken->setClient($client);
    $accessToken->setUserIdentifier($userIdentifier);
    $accessToken->setExpiryDateTime((new \DateTime())->add($accessTokenTTL));

    foreach ($scopes as $scope) {
        $accessToken->addScope($scope);
    }

    while ($maxGenerationAttempts-- > 0) {
        $accessToken->setIdentifier($this->generateUniqueIdentifier());
        try {
            $this->accessTokenRepository->persistNewAccessToken($accessToken);

            return $accessToken;
        } catch (UniqueTokenIdentifierConstraintViolationException $e) {
            if ($maxGenerationAttempts === 0) {
                throw $e;
            }
        }
    }
}

public function getNewToken(ClientEntityInterface $clientEntity, array $scopes, $userIdentifier = null)
{
    return new AccessToken($userIdentifier, $scopes);
}

protected function generateUniqueIdentifier($length = 40)
{
    try {
        return bin2hex(random_bytes($length));
        // @codeCoverageIgnoreStart
    } catch (\TypeError $e) {
        throw OAuthServerException::serverError('An unexpected error has occurred');
    } catch (\Error $e) {
        throw OAuthServerException::serverError('An unexpected error has occurred');
    } catch (\Exception $e) {
        // If you get this message, the CSPRNG failed hard.
        throw OAuthServerException::serverError('Could not generate a random string');
    }
    // @codeCoverageIgnoreEnd
}

public function persistNewAccessToken(AccessTokenEntityInterface $accessTokenEntity)
{
    $this->tokenRepository->create([
        'id' => $accessTokenEntity->getIdentifier(),
        'user_id' => $accessTokenEntity->getUserIdentifier(),
        'client_id' => $accessTokenEntity->getClient()->getIdentifier(),
        'scopes' => $this->scopesToArray($accessTokenEntity->getScopes()),
        'revoked' => false,
        'created_at' => new DateTime,
        'updated_at' => new DateTime,
        'expires_at' => $accessTokenEntity->getExpiryDateTime(),
    ]);

    $this->events->dispatch(new AccessTokenCreated(
        $accessTokenEntity->getIdentifier(),
        $accessTokenEntity->getUserIdentifier(),
        $accessTokenEntity->getClient()->getIdentifier()
    ));
}
```

类似地，还有生成 `refresh_token`：

```php
protected function issueRefreshToken(AccessTokenEntityInterface $accessToken)
{
    $maxGenerationAttempts = self::MAX_RANDOM_TOKEN_GENERATION_ATTEMPTS;

    $refreshToken = $this->refreshTokenRepository->getNewRefreshToken();
    $refreshToken->setExpiryDateTime((new \DateTime())->add($this->refreshTokenTTL));
    $refreshToken->setAccessToken($accessToken);

    while ($maxGenerationAttempts-- > 0) {
        $refreshToken->setIdentifier($this->generateUniqueIdentifier());
        try {
            $this->refreshTokenRepository->persistNewRefreshToken($refreshToken);

            return $refreshToken;
        } catch (UniqueTokenIdentifierConstraintViolationException $e) {
            if ($maxGenerationAttempts === 0) {
                throw $e;
            }
        }
    }
}
```

#### BearerToken 

为了加强安全性，根据 `OAuth2` 规范，`access_token` 与 `refresh_token` 需要利用 `Bearer Token` 的方式给出，`access token` 会被转化为 `JWT`，`refresh token` 会被加密:

```php
public function generateHttpResponse(ResponseInterface $response)
{
    $expireDateTime = $this->accessToken->getExpiryDateTime()->getTimestamp();

    $jwtAccessToken = $this->accessToken->convertToJWT($this->privateKey);

    $responseParams = [
        'token_type'   => 'Bearer',
        'expires_in'   => $expireDateTime - (new \DateTime())->getTimestamp(),
        'access_token' => (string) $jwtAccessToken,
    ];
I 
    if ($this->refreshToken instanceof RefreshTokenEntityInterface) {
        $refreshToken = $this->encrypt(
            json_encode(
                [
                    'client_id'        => $this->accessToken->getClient()->getIdentifier(),
                    'refresh_token_id' => $this->refreshToken->getIdentifier(),
                    'access_token_id'  => $this->accessToken->getIdentifier(),
                    'scopes'           => $this->accessToken->getScopes(),
                    'user_id'          => $this->accessToken->getUserIdentifier(),
                    'expire_time'      => $this->refreshToken->getExpiryDateTime()->getTimestamp(),
                ]
            )
        );

        $responseParams['refresh_token'] = $refreshToken;
    }

    $responseParams = array_merge($this->getExtraParams($this->accessToken), $responseParams);

    $response = $response
        ->withStatus(200)
        ->withHeader('pragma', 'no-cache')
        ->withHeader('cache-control', 'no-store')
        ->withHeader('content-type', 'application/json; charset=UTF-8');

    $response->getBody()->write(json_encode($responseParams));

    return $response;
}


```


### 刷新令牌

如果你的应用程序发放了短期的访问令牌，用户将需要通过在发出访问令牌时提供给他们的刷新令牌来刷新其访问令牌。该申请的 `url` 与申请令牌的链接相同，仅仅 `grant_type` 不同：

```php
$response = $http->post('http://your-app.com/oauth/token', [
    'form_params' => [
        'grant_type' => 'refresh_token',
        'refresh_token' => 'the-refresh-token',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => '',
    ],
]);

return json_decode((string) $response->getBody(), true);
```

```php
public function respondToAccessTokenRequest(
    ServerRequestInterface $request,
    ResponseTypeInterface $responseType,
    \DateInterval $accessTokenTTL
) {
    // Validate request
    $client = $this->validateClient($request);
    $oldRefreshToken = $this->validateOldRefreshToken($request, $client->getIdentifier());
    $scopes = $this->validateScopes($this->getRequestParameter(
        'scope',
        $request,
        implode(self::SCOPE_DELIMITER_STRING, $oldRefreshToken['scopes']))
    );

    // The OAuth spec says that a refreshed access token can have the original scopes or fewer so ensure
    // the request doesn't include any new scopes
    foreach ($scopes as $scope) {
        if (in_array($scope->getIdentifier(), $oldRefreshToken['scopes']) === false) {
            throw OAuthServerException::invalidScope($scope->getIdentifier());
        }
    }

    // Expire old tokens
    $this->accessTokenRepository->revokeAccessToken($oldRefreshToken['access_token_id']);
    $this->refreshTokenRepository->revokeRefreshToken($oldRefreshToken['refresh_token_id']);

    // Issue and persist new tokens
    $accessToken = $this->issueAccessToken($accessTokenTTL, $client, $oldRefreshToken['user_id'], $scopes);
    $refreshToken = $this->issueRefreshToken($accessToken);

    // Inject tokens into response
    $responseType->setAccessToken($accessToken);
    $responseType->setRefreshToken($refreshToken);

    return $responseType;
}
```