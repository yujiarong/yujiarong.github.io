---
title: Laravel Passport Api 认证
categories: PHP
tag: passport
abbrlink: 12855
date: 2019-03-15 00:00:00
---

此文用来梳理如何在Laravel中使用Passport的Personal Access Token 来做Api 用户认证。
版本: `Laravel`:`5.8` ,`Passport`:`~7.0`

### 安装

```bash 
#第一步 新建一个Laravel应用，下文例子是在已有项目中截出，可能会含有一些不必要的代码。
#记得配置一下 .env中的数据库
laravel new auth

#第二步 安装 Laravel Passport 包
composer require laravel/passport

#第三步 运行数据库迁移
php artisan migrate

# 生成秘钥
php artisan passport:install

#发布配置文件 只需要选择 passport-config 就好了，个人令牌不需要前端组件
php artisan vendor:publish

```

### 配置

将 Laravel\Passport\HasApiTokens trait 添加到你的 App\User 模型中。这个 trait 会为模型添加一系列助手函数用来验证用户的秘钥和作用域
```php 
<?php

namespace App\Models;

use Illuminate\Notifications\Notifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Permission\Traits\HasRoles;
use App\Models\Traits\ActionLogger;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use Notifiable;
    use HasRoles; 
    use ActionLogger;
    use HasApiTokens; //passport只需要加这个
}
```

在 AuthServiceProvider 中的 boot 方法中调用 Passport::routes 方法。这个方法会注册必要的路由去颁发访问令牌，撤销访问令牌，客户端和个人令牌：

```php 
<?php

namespace App\Providers;

use Laravel\Passport\Passport;
use Illuminate\Support\Facades\Gate;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The policy mappings for the application.
     *
     * @var array
     */
    protected $policies = [
        'App\Model' => 'App\Policies\ModelPolicy',
    ];

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();
        
        Passport::personalAccessTokensExpireIn(now()->addMinutes( config('passport.personal_access_tokens_expirein') ));
    }
}
```

在 config/auth.php 配置文件中，设置 api 权限认证守卫的 driver 选项为 passport。当需要权限认证的 API 请求进来时会告诉你的应用去使用 Passport 的 TokenGuard。
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

配置 Api 路由
在 routes/api.php 设置一些测试路由
```php 
Route::group([
    'prefix' => 'auth'
], function () {
    Route::post('login', 'Api\AuthController@login');
    Route::post('register', 'Api\AuthController@register');

    Route::group([
      'middleware' => 'auth:api'
    ], function() {
        Route::get('logout', 'Api\AuthController@logout');
        Route::get('user', 'Api\AuthController@user');
    });
});
```

### 控制器逻辑
在Controllers目录中新建Api目录，并且新建 ApiController.php 作为所有Api控制器的基类。Api响应格式统一在这个基类中处理，这里就先简单一点，只定义`responseJson`方法

```php 
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;

class ApiController extends Controller
{

    /**
     * 正常状态使用
     *
     * @param int    $code
     * @param string $msg
     * @param array  $data
     * @param array  $extFields
     * @return \Illuminate\Http\JsonResponse
     */
    function responseJson($code = 200, $msg = 'success', $data = [], $extFields = [])
    {
        $responseData = compact('code', 'msg', 'data');
        $responseData = array_merge($responseData, $extFields);
        return response()->json($responseData);
    }

}

```

创建控制器 App\Http\Controllers\Api\AuthController，处理Api的登录,注册,注销逻辑
这里接口字段验证使用了Request类，资源访问使用了Resource。具体可以看Laravel文档，也可以改成简单的写法不影响认证逻辑
```php 
<?php

namespace App\Http\Controllers\Api;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use App\Http\Requests\LoginRequest;
use App\Http\Requests\RegisterUserRequest;
use Illuminate\Support\Facades\Auth;
use App\Http\Resources\UserResource;
use Carbon\Carbon;

class AuthController extends ApiController
{
    /**
     * 登录的接口
     *
     * @param LoginRequest $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function login(LoginRequest $request)
    {
        $credentials = request(['email', 'password']);
        if( !Auth::attempt($credentials) ){
            return $this->responseJson(401,'账号密码错误');
        }

        $user = $request->user();

        $tokenResult = $user->createToken('Personal Access Token');
        $token = $tokenResult->token;
        $data  = [
            'access_token' => $tokenResult->accessToken,
            'token_type'   => 'Bearer',
            'expires_at'   => Carbon::parse(
                $tokenResult->token->expires_at
            )->toDateTimeString()
        ];
        return $this->responseJson(200, '登录成功',$data );

    }

    /**
     * 注销的接口
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function logout()
    {
        auth('api')->user()->token()->delete();
        return $this->responseJson(200, '退出成功' );
    }

    /**
     * 注册的接口
     *
     * @param RegisterUserRequest $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function register(RegisterUserRequest $request)
    {
        $name     = $request->input('name');
        $email    = $request->input('email');
        $password = $request->input('password');
        $user = new User();
        $user->name     = $name;
        $user->email    = $email;
        $user->password = bcrypt($password);
        $user->save();
        $tokenResult = $user->createToken('Personal Access Token');
        $token = $tokenResult->token;
        $data  = [
                'access_token'  =>  $tokenResult->accessToken,
                'token_type'    => 'Bearer',
                'expires_at'    =>  Carbon::parse(
                                        $tokenResult->token->expires_at
                                    )->toDateTimeString(),
                'user'          => new UserResource($user)
        ];
        return $this->responseJson(200, '注册成功',$data );
    }

    public function user(Request $request)
    {
        return response()->json($request->user());
    }


}

```

### 处理unauthenticated响应
登录失败Laravel会自动抛出异常，然后在Illuminate\Foundation\Exceptions\Handler中捕获此异常并且做出响应

```php
    /**
     * Convert an authentication exception into a response.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Illuminate\Auth\AuthenticationException  $exception
     * @return \Symfony\Component\HttpFoundation\Response
     */
    protected function unauthenticated($request, AuthenticationException $exception)
    {
        return $request->expectsJson()
                    ? response()->json(['message' => $exception->getMessage()], 401)
                    : redirect()->guest($exception->redirectTo() ?? route('login'));
    }
```

可以直接在App\Exceptions\Hander 中重写此方法，自定义你unauthenticated的响应格式，比如简单点如下:
```php

    protected function unauthenticated($request, AuthenticationException $exception)
    {
        return $request->expectsJson()
                    ? response()->json([ 'code'=> 300, 'message' => $exception->getMessage()], 401)
                    : redirect()->guest(route('login'));
    }
```

### 使用postman 测试接口
使用postman的时候 在header中加入`X-Requested-With:XMLHttpRequest`,`Content-Type: application/json`
php artisan serve  启动服务
1. 注册接口 
![regiser](/medias/post/laravel-passport-regiser.png)
> 注册后直接生成accessToken并返回，这样注册后就不需要再次登陆了
> accessToken生成之后可以在 `oauth_access_tokens`中查看

2. 登录接口 
![login](/medias/post/laravel-passport-login.png)
> 登录成功之后就可以将返回的access_token和token_type组合放到header去取请求需要认证的接口,比如请求user信息接口

3. 注销接口 
![logout](/medias/post/laravel-passport-logout.png)
>注销的时候可以做成全部token注销，也可以做成此次登录的token注销，看需要。其实就是删除 `oauth_access_tokens`中此用户的token记录

4. 获取用户信息接口
![user](/medias/post/laravel-psssport-user.png)

### passport  个人令牌过期时间设置

在 AuthServiceProvider 设置如下:
```php
Passport::personalAccessTokensExpireIn(now()->addMinutes( config('passport.personal_access_tokens_expirein') ));
```
>早期的passport版本可能不支持此方法，在AppServiceProvider的boot方法中加入下面的代码，时间格式使用的是DateInterval

```php
$this->app->get(AuthorizationServer::class)
    ->enableGrantType(new PersonalAccessGrant(), new \DateInterval('PT1M'));
```
### Passport 过期token删除
使用事件来清理 `oauth_access_tokens`表中过期的token

> 将事件注册到 EventServiceProvider 中，代码如下：

```php 
protected $listen = [
    'Laravel\Passport\Events\AccessTokenCreated' => [
        'App\Listeners\Auth\RevokeOldTokens',
    ],
];
```


> 新增RevokeOldTokens Listener，代码如下：

```php 
<?php

namespace App\Listeners\Auth;
use Laravel\Passport\Token;
use Laravel\Passport\Events\AccessTokenCreated;

class RevokeOldTokens
{
    /**
     * Handle the event.
     *
     * @param  AccessTokenCreated  $event
     * @return void
     */
    public function handle(AccessTokenCreated $event)
    {
        Token::where('id', '!=', $event->tokenId)
            ->where('user_id', $event->userId)
            ->where('client_id', $event->clientId)
            ->where('expires_at', '<', now())
            ->orWhere('revoked', true)
            ->delete();
    }

}
```

> 至此Passport 个人访问令牌 来认证Api 就可以正常使用了
> 如果要使用Passport的密码授权令牌可以参考这个这个 [Passport 密码授权令牌](https://learnku.com/articles/6976/laravel-55-uses-passport-to-implement-auth-authentication)