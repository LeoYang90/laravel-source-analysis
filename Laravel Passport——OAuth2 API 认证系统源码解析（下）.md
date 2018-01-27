# Laravel Passport——OAuth2 API 认证系统源码解析（下）

## 隐式授权
隐式授权类似于授权码授权，但是它只令牌将返回给客户端而不交换授权码。这种授权最常用于无法安全存储客户端凭据的 JavaScript 或移动应用程序。通过调用 `AuthServiceProvider` 中的 `enableImplicitGrant` 方法来启用这种授权：

```php
public function boot()
{
    $this->registerPolicies();

    Passport::routes();

    Passport::enableImplicitGrant();
}
```

调用上面方法开启授权后，开发者可以使用他们的客户端 ID 从应用程序请求访问令牌。接入的应用程序应该向你的应用程序的 /oauth/authorize 路由发出重定向请求，如下所示：

```php
Route::get('/redirect', function () {
    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://example.com/callback',
        'response_type' => 'token',
        'scope' => '',
    ]);

    return redirect('http://your-app.com/oauth/authorize?'.$query);
});
```

首先仍然是验证授权请求的合法性，其流程与授权码模式基本一致：

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

    $redirectUri = $this->getQueryStringParameter('redirect_uri', $request);

    $scopes = $this->validateScopes(
        $this->getQueryStringParameter('scope', $request, $this->defaultScope),
        is_array($client->getRedirectUri())
            ? $client->getRedirectUri()[0]
            : $client->getRedirectUri()
    );

    // Finalize the requested scopes
    $finalizedScopes = $this->scopeRepository->finalizeScopes(
        $scopes,
        $this->getIdentifier(),
        $client
    );

    $stateParameter = $this->getQueryStringParameter('state', $request);

    $authorizationRequest = new AuthorizationRequest();
    $authorizationRequest->setGrantTypeId($this->getIdentifier());
    $authorizationRequest->setClient($client);
    $authorizationRequest->setRedirectUri($redirectUri);
    $authorizationRequest->setState($stateParameter);
    $authorizationRequest->setScopes($finalizedScopes);

    return $authorizationRequest;
}
```

接着，当用户同意授权之后，就要直接返回 `access_token`，`League OAuth2` 直接将令牌放入 `JWT` 中发送回第三方客户端,值得注意的是依据 `OAuth2` 标准，参数都是以 `location hash` 的形式返回的，间隔符是 `#`，而不是 `?`:

```php
public function __construct(\DateInterval $accessTokenTTL, $queryDelimiter = '#')
{
    $this->accessTokenTTL = $accessTokenTTL;
    $this->queryDelimiter = $queryDelimiter;
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

    // The user approved the client, redirect them back with an access token
    if ($authorizationRequest->isAuthorizationApproved() === true) {
        $accessToken = $this->issueAccessToken(
            $this->accessTokenTTL,
            $authorizationRequest->getClient(),
            $authorizationRequest->getUser()->getIdentifier(),
            $authorizationRequest->getScopes()
        );

        $response = new RedirectResponse();
        $response->setRedirectUri(
            $this->makeRedirectUri(
                $finalRedirectUri,
                [
                    'access_token' => (string) $accessToken->convertToJWT($this->privateKey),
                    'token_type'   => 'Bearer',
                    'expires_in'   => $accessToken->getExpiryDateTime()->getTimestamp() - (new \DateTime())->getTimestamp(),
                    'state'        => $authorizationRequest->getState(),
                ],
                $this->queryDelimiter
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

这个用于构建 `jwt` 的私钥就是 `oauth-private.key`，我们知道，`jwt` 一般有三个部分组成：`header`、`claim`、`sign`, 用于 `oauth2` 的 `jwt` 中 `claim` 主要构成有：

- aud  客户端 id
- jti  access_token 随机码
- iat  生成时间
- nbf  拒绝接受 jwt 时间
- exp  access_token 失效时间
- sub  用户 id

具体可以参考 : [JSON Web Token (JWT) draft-ietf-oauth-json-web-token-32](https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32)

```php
public function convertToJWT(CryptKey $privateKey)
{
    return (new Builder())
        ->setAudience($this->getClient()->getIdentifier())
        ->setId($this->getIdentifier(), true)
        ->setIssuedAt(time())
        ->setNotBefore(time())
        ->setExpiration($this->getExpiryDateTime()->getTimestamp())
        ->setSubject($this->getUserIdentifier())
        ->set('scopes', $this->getScopes())
        ->sign(new Sha256(), new Key($privateKey->getKeyPath(), $privateKey->getPassPhrase()))
        ->getToken();
}

public function __construct(
    Encoder $encoder = null,
    ClaimFactory $claimFactory = null
) {
    $this->encoder = $encoder ?: new Encoder();
    $this->claimFactory = $claimFactory ?: new ClaimFactory();
    $this->headers = ['typ'=> 'JWT', 'alg' => 'none'];
    $this->claims = [];
}

public function setAudience($audience, $replicateAsHeader = false)
{
    return $this->setRegisteredClaim('aud', (string) $audience, $replicateAsHeader);
}

public function setId($id, $replicateAsHeader = false)
{
    return $this->setRegisteredClaim('jti', (string) $id, $replicateAsHeader);
}

public function setIssuedAt($issuedAt, $replicateAsHeader = false)
{
    return $this->setRegisteredClaim('iat', (int) $issuedAt, $replicateAsHeader);
}

public function setNotBefore($notBefore, $replicateAsHeader = false)
{
    return $this->setRegisteredClaim('nbf', (int) $notBefore, $replicateAsHeader);
}

public function setExpiration($expiration, $replicateAsHeader = false)
{
    return $this->setRegisteredClaim('exp', (int) $expiration, $replicateAsHeader);
}

public function setSubject($subject, $replicateAsHeader = false)
{
    return $this->setRegisteredClaim('sub', (string) $subject, $replicateAsHeader);
}

public function sign(Signer $signer, $key)
{
    $signer->modifyHeader($this->headers);

    $this->signature = $signer->sign(
        $this->getToken()->getPayload(),
        $key
    );

    return $this;
}

public function getToken()
{
    $payload = [
        $this->encoder->base64UrlEncode($this->encoder->jsonEncode($this->headers)),
        $this->encoder->base64UrlEncode($this->encoder->jsonEncode($this->claims))
    ];

    if ($this->signature !== null) {
        $payload[] = $this->encoder->base64UrlEncode($this->signature);
    }

    return new Token($this->headers, $this->claims, $this->signature, $payload);
}
```

根据 JWT 的生成方法，签名部分 `signature` 是 `header` 与 `claim` 进行 `base64` 编码后再加密的结果。

## 客户端模式

客户端凭据授权适用于机器到机器的认证。例如，你可以在通过 API 执行维护任务中使用此授权。要使用这种授权，你首先需要在 app/Http/Kernel.php 的 routeMiddleware 变量中添加新的中间件：

```php
protected $routeMiddleware = [
    'client' => CheckClientCredentials::class,
];

Route::get('/user', function(Request $request) {
    ...
})->middleware('client');

```

接下来通过向 oauth/token 接口发出请求来获取令牌:

```php
$response = $guzzle->post('http://your-app.com/oauth/token', [
    'form_params' => [
        'grant_type' => 'client_credentials',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => 'your-scope',
    ],
]);

echo json_decode((string) $response->getBody(), true);

```

客户端模式类似于授权码模式的后一部分，利用客户端 id 与客户端密码来获取 `access_token`：

```php
public function respondToAccessTokenRequest(
    ServerRequestInterface $request,
    ResponseTypeInterface $responseType,
    \DateInterval $accessTokenTTL
) {
    // Validate request
    $client = $this->validateClient($request);
    $scopes = $this->validateScopes($this->getRequestParameter('scope', $request, $this->defaultScope));

    // Finalize the requested scopes
    $finalizedScopes = $this->scopeRepository->finalizeScopes($scopes, $this->getIdentifier(), $client);

    // Issue and persist access token
    $accessToken = $this->issueAccessToken($accessTokenTTL, $client, null, $finalizedScopes);

    // Inject access token into response type
    $responseType->setAccessToken($accessToken);

    return $responseType;
}
```

类似于授权码模式，`access_token` 的发放也是通过 `Bearer Token` 中存放 JWT。 


## 密码模式

OAuth2 密码授权机制可以让你自己的客户端（如移动应用程序）邮箱地址或者用户名和密码获取访问令牌。如此一来你就可以安全地向自己的客户端发出访问令牌，而不需要遍历整个 OAuth2 授权代码重定向流程。

创建密码授权的客户端后，就可以通过向用户的电子邮件地址和密码向 /oauth/token 路由发出 POST 请求来获取访问令牌。而该路由已经由 Passport::routes 方法注册，因此不需要手动定义它。如果请求成功，会在服务端返回的 JSON 响应中收到一个 access_token 和 refresh_token：

```php
$response = $http->post('http://your-app.com/oauth/token', [
    'form_params' => [
        'grant_type' => 'password',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'username' => 'taylor@laravel.com',
        'password' => 'my-password',
        'scope' => '',
    ],
]);

return json_decode((string) $response->getBody(), true);

```

只要用用户名与密码来验证合法性就可以发放 `access_token` 与 `refresh_token`：

```php
public function respondToAccessTokenRequest(
    ServerRequestInterface $request,
    ResponseTypeInterface $responseType,
    \DateInterval $accessTokenTTL
) {
    // Validate request
    $client = $this->validateClient($request);
    $scopes = $this->validateScopes($this->getRequestParameter('scope', $request, $this->defaultScope));
    $user = $this->validateUser($request, $client);

    // Finalize the requested scopes
    $finalizedScopes = $this->scopeRepository->finalizeScopes($scopes, $this->getIdentifier(), $client, $user->getIdentifier());

    // Issue and persist new tokens
    $accessToken = $this->issueAccessToken($accessTokenTTL, $client, $user->getIdentifier(), $finalizedScopes);
    $refreshToken = $this->issueRefreshToken($accessToken);

    // Inject tokens into response
    $responseType->setAccessToken($accessToken);
    $responseType->setRefreshToken($refreshToken);

    return $responseType;
}


protected function validateUser(ServerRequestInterface $request, ClientEntityInterface $client)
{
    $username = $this->getRequestParameter('username', $request);
    if (is_null($username)) {
        throw OAuthServerException::invalidRequest('username');
    }

    $password = $this->getRequestParameter('password', $request);
    if (is_null($password)) {
        throw OAuthServerException::invalidRequest('password');
    }

    $user = $this->userRepository->getUserEntityByUserCredentials(
        $username,
        $password,
        $this->getIdentifier(),
        $client
    );
    if ($user instanceof UserEntityInterface === false) {
        $this->getEmitter()->emit(new RequestEvent(RequestEvent::USER_AUTHENTICATION_FAILED, $request));

        throw OAuthServerException::invalidCredentials();
    }

    return $user;
}

```
## 路由保护

Passport 包含一个 验证保护机制 可以验证请求中传入的访问令牌。配置 api 的看守器使用 passport 驱动程序后，只需要在需要有效访问令牌的任何路由上指定 auth:api 中间件：

```php
Route::get('/user', function () {
    //
})->middleware('auth:api');
```

当调用 Passport 保护下的路由时，接入的 API 应用需要将访问令牌作为 Bearer 令牌放在请求头 Authorization 中。例如，使用 Guzzle HTTP 库时：

```php
$response = $client->request('GET', '/api/user', [
    'headers' => [
        'Accept' => 'application/json',
        'Authorization' => 'Bearer '.$accessToken,
    ],
]);
```


## auth:api 中间件

当我们已经配置完成 `Passport` 的四种模式并拿到 `access_token` 之后，我们就可以利用令牌去资源服务器获取数据了。资源服务器最常用的校验令牌的中间件就是 `auth:api`，中间件是 `auth`，`api` 是中间件的参数：

```php

'auth' => \Illuminate\Auth\Middleware\Authenticate::class,

```
这个中间件是验证登录状态的常用中间件：

```php

class Authenticate
{
    public function __construct(Auth $auth)
    {
        $this->auth = $auth;
    }
    
    public function handle($request, Closure $next, ...$guards)
    {
        $this->authenticate($guards);

        return $next($request);
    }


    protected function authenticate(array $guards)
    {
        if (empty($guards)) {
            return $this->auth->authenticate();
        }

        foreach ($guards as $guard) {
            if ($this->auth->guard($guard)->check()) {
                return $this->auth->shouldUse($guard);
            }
        }

        throw new AuthenticationException('Unauthenticated.', $guards);
    }
}

```

我们的参数 `api` 就是上面的 `guards`，`Auth` 是 `laravel` 自带的登录校验服务：

```php

class AuthManager implements FactoryContract
{
    public function guard($name = null)
    {
        $name = $name ?: $this->getDefaultDriver();

        return $this->guards[$name] ?? $this->guards[$name] = $this->resolve($name);
    }

    protected function resolve($name)
    {
        $config = $this->getConfig($name);

        if (is_null($config)) {
            throw new InvalidArgumentException("Auth guard [{$name}] is not defined.");
        }

        if (isset($this->customCreators[$config['driver']])) {
            return $this->callCustomCreator($name, $config);
        }

        $driverMethod = 'create'.ucfirst($config['driver']).'Driver';

        if (method_exists($this, $driverMethod)) {
            return $this->{$driverMethod}($name, $config);
        }

        throw new InvalidArgumentException("Auth guard driver [{$name}] is not defined.");
    }

}

```

文档告诉我们，若想要使用 `passport` 服务，我们的 `config/auth` 文件需要如此配置：

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

可以看出，`driver` 就是 `passport`，我们在启动 `passport` 服务的时候曾经注册过一个 `Guard`：

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

因此，`passport` 使用的就是这个 `TokenGuard`：

```php
class TokenGuard
{
    public function __construct(ResourceServer $server,
                                UserProvider $provider,
                                TokenRepository $tokens,
                                ClientRepository $clients,
                                Encrypter $encrypter)
    {
        $this->server = $server;
        $this->tokens = $tokens;
        $this->clients = $clients;
        $this->provider = $provider;
        $this->encrypter = $encrypter;
    }
    
    public function user(Request $request)
    {
        if ($request->bearerToken()) {
            return $this->authenticateViaBearerToken($request);
        } elseif ($request->cookie(Passport::cookie())) {
            return $this->authenticateViaCookie($request);
        }
    }
}
```
可以看到，`TokenGuard` 支持两种 `Token` 的验证：`BearerToken` 与 `cookie`。

我们首先看 `BearerToken`:

```php
public function bearerToken()
{
    $header = $this->header('Authorization', '');

    if (Str::startsWith($header, 'Bearer ')) {
        return Str::substr($header, 7);
    }
}

protected function authenticateViaBearerToken($request)
{
    $psr = (new DiactorosFactory)->createRequest($request);

    try {
        $psr = $this->server->validateAuthenticatedRequest($psr);

        $user = $this->provider->retrieveById(
            $psr->getAttribute('oauth_user_id')
        );

        if (! $user) {
            return;
        }

        $token = $this->tokens->find(
            $psr->getAttribute('oauth_access_token_id')
        );

        $clientId = $psr->getAttribute('oauth_client_id');

        if ($this->clients->revoked($clientId)) {
            return;
        }

        return $token ? $user->withAccessToken($token) : null;
    } catch (OAuthServerException $e) {
        return Container::getInstance()->make(
            ExceptionHandler::class
        )->report($e);
    }
}

```

首先，需要验证请求的合法性：

```php
class ResourceServer
{
    public function validateAuthenticatedRequest(ServerRequestInterface $request)
    {
        return $this->getAuthorizationValidator()->validateAuthorization($request);
    }

    protected function getAuthorizationValidator()
    {
        if ($this->authorizationValidator instanceof AuthorizationValidatorInterface === false) {
            $this->authorizationValidator = new BearerTokenValidator($this->accessTokenRepository);
        }

        $this->authorizationValidator->setPublicKey($this->publicKey);

        return $this->authorizationValidator;
    }

}
```

`BearerTokenValidator` 专门用于验证 `BearerToken` 的合法性：

```php
class BearerTokenValidator implements AuthorizationValidatorInterface
{
    public function validateAuthorization(ServerRequestInterface $request)
    {
        if ($request->hasHeader('authorization') === false) {
            throw OAuthServerException::accessDenied('Missing "Authorization" header');
        }

        $header = $request->getHeader('authorization');
        $jwt = trim(preg_replace('/^(?:\s+)?Bearer\s/', '', $header[0]));

        try {
            // Attempt to parse and validate the JWT
            $token = (new Parser())->parse($jwt);
            if ($token->verify(new Sha256(), $this->publicKey->getKeyPath()) === false) {
                throw OAuthServerException::accessDenied('Access token could not be verified');
            }

            // Ensure access token hasn't expired
            $data = new ValidationData();
            $data->setCurrentTime(time());

            if ($token->validate($data) === false) {
                throw OAuthServerException::accessDenied('Access token is invalid');
            }

            // Check if token has been revoked
            if ($this->accessTokenRepository->isAccessTokenRevoked($token->getClaim('jti'))) {
                throw OAuthServerException::accessDenied('Access token has been revoked');
            }

            // Return the request with additional attributes
            return $request
                ->withAttribute('oauth_access_token_id', $token->getClaim('jti'))
                ->withAttribute('oauth_client_id', $token->getClaim('aud'))
                ->withAttribute('oauth_user_id', $token->getClaim('sub'))
                ->withAttribute('oauth_scopes', $token->getClaim('scopes'));
        } catch (\InvalidArgumentException $exception) {
            // JWT couldn't be parsed so return the request as is
            throw OAuthServerException::accessDenied($exception->getMessage());
        } catch (\RuntimeException $exception) {
            //JWR couldn't be parsed so return the request as is
            throw OAuthServerException::accessDenied('Error while decoding to JSON');
        }
    }
}
```

通过 `passport` 拿到的 `access_token` 都是 `JWT` 格式的，因此首先第一步需要将 `JWT` 解析：

```php
class Parser
{
    public function parse($jwt)
    {
        $data = $this->splitJwt($jwt);
        $header = $this->parseHeader($data[0]);
        $claims = $this->parseClaims($data[1]);
        $signature = $this->parseSignature($header, $data[2]);

        foreach ($claims as $name => $value) {
            if (isset($header[$name])) {
                $header[$name] = $value;
            }
        }

        if ($signature === null) {
            unset($data[2]);
        }

        return new Token($header, $claims, $signature, $data);
    }
    
    protected function splitJwt($jwt)
    {
        if (!is_string($jwt)) {
            throw new InvalidArgumentException('The JWT string must have two dots');
        }

        $data = explode('.', $jwt);

        if (count($data) != 3) {
            throw new InvalidArgumentException('The JWT string must have two dots');
        }

        return $data;
    }
    
    protected function parseHeader($data)
    {
        $header = (array) $this->decoder->jsonDecode($this->decoder->base64UrlDecode($data));

        if (isset($header['enc'])) {
            throw new InvalidArgumentException('Encryption is not supported yet');
        }

        return $header;
    }
    
    protected function parseClaims($data)
    {
        $claims = (array) $this->decoder->jsonDecode($this->decoder->base64UrlDecode($data));

        foreach ($claims as $name => &$value) {
            $value = $this->claimFactory->create($name, $value);
        }

        return $claims;
    }
    
    protected function parseSignature(array $header, $data)
    {
        if ($data == '' || !isset($header['alg']) || $header['alg'] == 'none') {
            return null;
        }

        $hash = $this->decoder->base64UrlDecode($data);

        return new Signature($hash);
    }

}
```

获得 `JWT` 的三个部分之后，就要验证签名部分是否合法：

```php
class Token
{
    public function verify(Signer $signer, $key)
    {
        if ($this->signature === null) {
            throw new BadMethodCallException('This token is not signed');
        }

        if ($this->headers['alg'] !== $signer->getAlgorithmId()) {
            return false;
        }

        return $this->signature->verify($signer, $this->getPayload(), $key);
    }
}
```

验证通过之后，就要验证 `JWT` 各个部分是否合法：

```php
$data = new ValidationData();
$data->setCurrentTime(time());

public function __construct($currentTime = null)
{
    $currentTime = $currentTime ?: time();

    $this->items = [
        'jti' => null,
        'iss' => null,
        'aud' => null,
        'sub' => null,
        'iat' => $currentTime,
        'nbf' => $currentTime,
        'exp' => $currentTime
    ];
}
            
            
public function validate(ValidationData $data)
{
    foreach ($this->getValidatableClaims() as $claim) {
        if (!$claim->validate($data)) {
            return false;
        }
    }

    return true;
}    

public function __construct(array $callbacks = [])
{
    $this->callbacks = array_merge(
        [
            'iat' => [$this, 'createLesserOrEqualsTo'],
            'nbf' => [$this, 'createLesserOrEqualsTo'],
            'exp' => [$this, 'createGreaterOrEqualsTo'],
            'iss' => [$this, 'createEqualsTo'],
            'aud' => [$this, 'createEqualsTo'],
            'sub' => [$this, 'createEqualsTo'],
            'jti' => [$this, 'createEqualsTo']
        ],
        $callbacks
    );
}
```

我们前面说过，

- aud 客户端 id
- jti access_token 随机码
- iat 生成时间
- nbf 拒绝接受 jwt 时间
- exp access_token 失效时间
- sub 用户 id

因此，`JWT` 的生成时间、拒绝接受时间、失效时间就会被验证完成。

接下来，还会验证最重要的 `access_token` ：

```php
if ($this->accessTokenRepository->isAccessTokenRevoked($token->getClaim('jti'))) {
    throw OAuthServerException::accessDenied('Access token has been revoked');
}

public function isAccessTokenRevoked($tokenId)
{
    return $this->tokenRepository->isAccessTokenRevoked($tokenId);
}

public function isAccessTokenRevoked($id)
{
    if ($token = $this->find($id)) {
        return $token->revoked;
    }

    return true;
}
```

接下来，`TokenGuard` 就会验证 `userid`、`clientid` 与 `access_token` 的合法性：

```php
$user = $this->provider->retrieveById(
    $psr->getAttribute('oauth_user_id')
);

if (! $user) {
    return;
}

$token = $this->tokens->find(
    $psr->getAttribute('oauth_access_token_id')
);

$clientId = $psr->getAttribute('oauth_client_id');

if ($this->clients->revoked($clientId)) {
    return;
}

return $token ? $user->withAccessToken($token) : null;


```

中间件验证完成。

## 客户端模式中间件 CheckClientCredentials

我们在上面可以看到 `auth:api` 中间件不仅验证 `access_token`，还会验证 `user_id`，对于客户端模式来说，由于 `JWT` 中并没有用户信息，因此 `passport` 专门存在中间件 `CheckClientCredentials` 来做非登录状态的校验。

```php
class CheckClientCredentials
{
    public function handle($request, Closure $next, ...$scopes)
    {
        $psr = (new DiactorosFactory)->createRequest($request);

        try {
            $psr = $this->server->validateAuthenticatedRequest($psr);
        } catch (OAuthServerException $e) {
            throw new AuthenticationException;
        }

        $this->validateScopes($psr, $scopes);

        return $next($request);
    }
}
```

## 使用 JavaScript 接入 API

在构建 API 时，如果能通过 JavaScript 应用接入自己的 API 将会给开发过程带来极大的便利。这种 API 开发方法允许你使用自己的应用程序的 API 和别人共享的 API。你的 Web 应用程序、移动应用程序、第三方应用程序以及可能在各种软件包管理器上发布的任何 SDK 都可能会使用相同的API。

通常，如果要从 JavaScript 应用程序中使用 API，则需要手动向应用程序发送访问令牌，并将其传递给应用程序。但是，Passport 有一个可以处理这个问题的中间件。将 CreateFreshApiToken 中间件添加到 web 中间件组就可以了：

```php
'web' => [
    // Other middleware...
    \Laravel\Passport\Http\Middleware\CreateFreshApiToken::class,
],
```
Passport 的这个中间件将会在你所有的对外请求中添加一个 laravel_token cookie。该 cookie 将包含一个加密后的 JWT ，Passport 将用来验证来自 JavaScript 应用程序的 API 请求。至此，你可以在不明确传递访问令牌的情况下向应用程序的 API 发出请求

```php
axios.get('/user')
    .then(response => {
        console.log(response.data);
    });
```
当使用上面的授权方法时，Axios 会自动带上 X-CSRF-TOKEN 请求头传递。另外，默认的 Laravel JavaScript 脚手架会让 Axios 发送 X-Requested-With 请求头:

```php
window.axios.defaults.headers.common = {
    'X-Requested-With': 'XMLHttpRequest',
};

```

## CreateFreshApiToken 中间件

```php
class CreateFreshApiToken
{
    public function handle($request, Closure $next, $guard = null)
    {
        $this->guard = $guard;

        $response = $next($request);

        if ($this->shouldReceiveFreshToken($request, $response)) {
            $response->withCookie($this->cookieFactory->make(
                $request->user($this->guard)->getKey(), $request->session()->token()
            ));
        }

        return $response;
    }
    
    public function make($userId, $csrfToken)
    {
        $config = $this->config->get('session');

        $expiration = Carbon::now()->addMinutes($config['lifetime']);

        return new Cookie(
            Passport::cookie(),
            $this->createToken($userId, $csrfToken, $expiration),
            $expiration,
            $config['path'],
            $config['domain'],
            $config['secure'],
            true
        );
    }
    
    protected function createToken($userId, $csrfToken, Carbon $expiration)
    {
        return JWT::encode([
            'sub' => $userId,
            'csrf' => $csrfToken,
            'expiry' => $expiration->getTimestamp(),
        ], $this->encrypter->getKey());
    }

    protected function shouldReceiveFreshToken($request, $response)
    {
        return $this->requestShouldReceiveFreshToken($request) &&
               $this->responseShouldReceiveFreshToken($response);
    }

    protected function requestShouldReceiveFreshToken($request)
    {
        return $request->isMethod('GET') && $request->user($this->guard);
    }
    
    protected function responseShouldReceiveFreshToken($response)
    {
        return $response instanceof Response && ! $this->alreadyContainsToken($response);
    }
}
```

这个中间件发出的 `JWT` 令牌仍然由 `auth:api` 来负责验证，我们前面说过，`TokenGuard` 负责两种令牌的验证，一种是 `BearerToken`, 另一种就是这个 `Cookie` :

```php
public function user(Request $request)
{
    if ($request->bearerToken()) {
        return $this->authenticateViaBearerToken($request);
    } elseif ($request->cookie(Passport::cookie())) {
        return $this->authenticateViaCookie($request);
    }
}

protected function authenticateViaCookie($request)
{
    try {
        $token = $this->decodeJwtTokenCookie($request);
    } catch (Exception $e) {
        return;
    }

    if (! $this->validCsrf($token, $request) ||
        time() >= $token['expiry']) {
        return;
    }

    if ($user = $this->provider->retrieveById($token['sub'])) {
        return $user->withAccessToken(new TransientToken);
    }
}

protected function decodeJwtTokenCookie($request)
{
    return (array) JWT::decode(
        $this->encrypter->decrypt($request->cookie(Passport::cookie())),
        $this->encrypter->getKey(), ['HS256']
    );
}

protected function validCsrf($token, $request)
{
    return isset($token['csrf']) && hash_equals(
        $token['csrf'], (string) $request->header('X-CSRF-TOKEN')
    );
}
```









