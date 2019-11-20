---
date: "2017-03-08"
title: Lumen中使用ValidatorMiddleware中间件处理请求参数验证
description: php, lumen, validator
slug: validator-middleware
categories:
- Web
tags:
- php
- lumen
comments: true
---

本次文章的缘由来之下面这段代码
```php
public function test()
{
    $message = [
        'object_id.required' => ErrorCode::PARAM_MISS,
    ];
    $validator = Validator::make($this->request->all(), [
        'object_id' => 'required',
    ], $message);
    if ($validator->fails()) {
        ...error
    }
    ...
}
```

上面这个场景应该应该大家都不会陌生，验证请求参数是一个非常基础的场景，当需要验证的参数较少时看上去也还能勉强接受，但如果接口接收的参数较多时就会变成这样😂

{{< image src="/assets/images/validator-middleware-1.png" alt="validator-middleware-1" position="center" >}}
（其实还有10+个参数的图，考虑到密集恐惧症患者的感受没有发🙂）好吧，没办法。谁让我们这个接口接收的参数多呢！怎么办，我也很无奈呀！

**思考**：真的没有办法吗？我review了最近写的几个controller，发现了一个共同点：每一个接口请求对应的处理的方法中，第一段都是处理对应的参数验证的逻辑，这段逻辑除了对应接口的验证规则不一样，其他都是调用相同的逻辑。其实都是重复的代码！怎么样可以把这些代码优化一下呢？

好，通过分析上面的问题，我的第一反应是引用中间件来统一处理这部分的逻辑。网上找了下相关的处理方式，并且参考了下Laravel FormRequest的实现，我设计了ValidatorMiddleware中间件来解决这个问题，代码如下。
```php
namespace App\Http\Middleware;

use Closure;

class ValidatorMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        list($controller, $method) = explode('@', $request->route()[1]['uses']);
        $class = str_replace('Controller', 'Request', $controller);
        if (class_exists($class)) {
            $messageRequest = new $class($request, $method);
            $messageRequest->validate();
        }

        return $next($request);
    }
}
```

```php
/** Request处理基类 **/
namespace App\Http\Requests;

use Illuminate\Http\Request;
use App\Exceptions\ResponseException;
use Validator;

class BaseRequest
{
    protected $input;
    protected $method;

    public function __construct(Request $request, String $method)
    {
        $this->input = $request->all();
        $this->method = $method;
    }

    public function rules()
    {
        return [];
    }

    public function messages()
    {
        return [];
    }

    public function validate()
    {
        $rules = array_get($this->rules(), $this->method, []);
        $messages = array_get($this->messages(), $this->method, []);
        $validator = Validator::make($this->input, $rules, $messages);
        if ($validator->fails()) {
            $err = $validator->errors()->all();
            throw new ResponseException(head($err));
        }

        return true;
    }
}
```

下面结合实际的例子来讲解:

> 请求url: /user/get?id=1 <br />
> 对应处理的controller@function: UserController@get <br />

```php
/** 那么对应的UserRequest */
namespace App\Http\Requests;

use App\ErrorCode;
use App\Http\Requests\BaseRequest;

class UserRequest extends BaseRequest
{
    public function rules()
    {
        return [
            'get' => [
                'id' => 'required|integer',
            ]
        ];
    }

    public function messages()
    {
        return [
            'get' => [
                'id.required' => ErrorCode::PARAM_MISS,
                'id.integer' => ErrorCode::PARAM_ERROR,
            ]
        ];
    }
}
```

当/user/get?id=1被访问时，首先会进入`ValidatorMiddleware->handle`方法中，`$request->route()[1]['uses']`会返回`App\Http\Controllers\UserController@get`, 所以代码先会去找有没有对应的`App\Http\Requests\UserRequest`, 如果有则会去执行它的validate方法。下面我们看UserRequest，它里面并没有validate，但是它继承了BaseRequest，所以我们再看BaseRequest的validate，这里的代码就比较熟悉了，就是我们之前调用Lumen的Validator类进行参数验证的逻辑了，由于在new UserRequest时指定了method参数，所以在验证时会去取方法对应的验证规则进行参数验证，如果失败则会抛出我们自定义的异常，最后，我们只要在`Exception\Handler`中处理自定义的异常就好了。

口水了一大堆，下面总结一下，核心思想如下：

> Controllers目录同级下会多出一个Requests目录，每个controller都会有对应的request类，request类中会有rules、messages（对应Validator类的验证规则和返回消息）俩个二维数组，一维的键值对应要处理的controller的方法，二维对应具体规则和消息。引用ValidatorMiddleware中间件统一处理接口请求，通过自定义异常类处理错误返回。

参考了Laravel China社区各位大神的设计，结合了下自己项目中的场景，设计了以上方案，欢迎大家拍砖。

---

**参考**

* [https://laravel-china.org/topics/3137/design-of-shared-verification-rule-layer](https://laravel-china.org/topics/3137/design-of-shared-verification-rule-layer )