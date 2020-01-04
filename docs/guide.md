
# 가이드

이 가이드는 미들웨어 사용법과 애플리케이션 구조에 대해서 다룬다.
예제의 미들웨어에서는 async 함수를 사용한다. 물론 commonFunction 또는 generatorFunction을 사용할 수도 있다.

## 목차

- [미들웨어 사용법](#미들웨어-사용법)
- [미들웨어 표준 사용법](#미들웨어-표준-사용법)
  - [미들웨어 옵션들](#미들웨어-옵션들)
  - [미들웨어 이름 지정](#미들웨어-이름-지정)
  - [koa-compose를 통해 다양한 미들웨어 조합하기](#koa-compose를-통해-다양한-미들웨어-조합하기)
  - [미들웨어 Response](#미들웨어-Response)
- [Async operations](#async-operations)
- [Debugging Koa](#debugging-koa)

# 미들웨어 사용법

  Koa 미들웨어는 (ctx, next) 같은 파라미터를 가진 `MiddlewareFunction` 반환하는 간단한 함수다. 미들웨어가 동작할 때, 반드시 `next()` 를 통해 다음 미들웨어로 갈 수 있다. 

만약 Koa를 통해 전파되는 request 소요 시간을 추적하고 싶다면 다음과 같이 `X-Response-Time`  헤더에 시간을 셋팅하면 된다.

```js
async function responseTime(ctx, next) {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
}

app.use(responseTime);
```

프론트엔드 개발자는 `next()` 전의 코드를 "capture" 단계로 `next()` 후의 코드를 "bubble" 단계로 생각할 수 있다.  아래의 이미지는 어떻게 async 함수로 request, reponse 구현을 스택 플로우로 활용할 수 있는 지 설명해준다.

![Koa middleware](/docs/middleware.gif)

   1. response time 생성
   2. Await를 통해 다음 미들웨어로
   3. 소요시간 계산을 위해 또 다른 time을 생성
   4. Await를 통해 다음 미들웨어로
   5. response body에 "Hello World" 셋팅
   6. 소요시간 계산
   7. 로그로 출력
   8. 응답시간 계산
   9. `X-Response-Time`  헤더 필드에 값 셋팅
   10. Koa로 이동하여 response 전달 

이제 우리는 Koa 미들웨어를 생성하는 방법에 대해 알아볼 것이다.

## 미들웨어 표준 사용법

이 섹션은 미들웨어 옵션, 디버깅 등을 위한 미들웨어 사용 방법에 대해 다룬다.

### 미들웨어 옵션들

공용 미들웨어를 생성할 때 옵션을 허용하여 편리하게 함수를 확장하여 미들웨어를 랩핑할 수 있다.  미들웨어에서 옵션을 허용하지 않을 수도 있는 데, 이는 좋은 표준이기도 하다.

여기 `logger` 미들웨어는 커스텀을 위해 `format`을 이용하여 값을 셋팅 후 문자열로 리턴한다.

```js
function logger(format) {
  format = format || ':method ":url"';

  return async function (ctx, next) {
    const str = format
      .replace(':method', ctx.method)
      .replace(':url', ctx.url);

    console.log(str);

    await next();
  };
}

app.use(logger());
app.use(logger(':method :url'));
```

### 미들웨어 이름 지정

미들웨어의 이름을 지정하는 것은 선택사항이나 이름을 사용하면 디버깅 목적으로 사용할 때 유용하다.

```js
function logger(format) {
  return async function logger(ctx, next) {

  };
}
```

### koa-compose를 통해 다양한 미들웨어 조합하기

[koa-compose](https://github.com/koajs/compose) 를 사용하면 여러 개의 미들웨어를 조립하여 하나의 미들웨어로 사용할 수 있다. 미들웨어를 재사용하거나 export할 때 유리하다. 

```js
const compose = require('koa-compose');

async function random(ctx, next) {
  if ('/random' == ctx.path) {
    ctx.body = Math.floor(Math.random() * 10);
  } else {
    await next();
  }
};

async function backwards(ctx, next) {
  if ('/backwards' == ctx.path) {
    ctx.body = 'sdrawkcab';
  } else {
    await next();
  }
}

async function pi(ctx, next) {
  if ('/pi' == ctx.path) {
    ctx.body = String(Math.PI);
  } else {
    await next();
  }
}

const all = compose([random, backwards, pi]);

app.use(all);
```

### 미들웨어 Response

미들웨어는 요청에 대한 응답을 위해 `next()`를 생략할지도 모른다. 일반적으로 `next()`는 미들웨어의 라우팅을 결정하나 다음과 같은 작업을 한다.
예를 들어 아래의 코드는 요청에 대해 "two"를 응답하나 3개의 미들웨어 모두 실행된다. 3개의 미들웨어에 response를 조작할 수 있는 기회가 주어지는 것이다.

```js
app.use(async function (ctx, next) {
  console.log('>> one');
  await next();
  console.log('<< one');
});

app.use(async function (ctx, next) {
  console.log('>> two');
  ctx.body = 'two';
  await next();
  console.log('<< two');
});

app.use(async function (ctx, next) {
  console.log('>> three');
  await next();
  console.log('<< three');
});
```
  
  아래의 코드는 두번째 미들웨어에서 `next()`를 생략했고 response로 "two"를 셋팅하여 세번째 미들웨어는 무시될 것이다.

```js
app.use(async function (ctx, next) {
  console.log('>> one');
  await next();
  console.log('<< one');
});

app.use(async function (ctx, next) {
  console.log('>> two');
  ctx.body = 'two';
  console.log('<< two');
});

app.use(async function (ctx, next) {
  console.log('>> three');
  await next();
  console.log('<< three');
});
```

가장 마지막 미들웨어에서 `next()`를 실행하면 noop function, 아무 동작을 하지 않으며 미들웨어가 스택의 어느 곳에서나 올바르게 구성할 수 있다.

## Async operations

Async 함수와 promise 기반인 Koa는 non-blocking sequential code를 허용한다. 예를 들어 이 미들웨어는 `./docs` 디렉토리에서 파일 이름들을 읽어온 후  parallel, 병렬로 각각의 마크다운 파일 형태의 내용을 읽어 body에 결과를 join 하여 셋팅한다.

> non-blocking이란, 어떤 쓰레드에서 오류가 발생하거나 멈추었을 때 다른 쓰레드에게 영향을 끼치지 않도록 만드는 방법들을 말한다.

> parallel, 병렬이라는 뜻처럼 데이터를 한번에 여러 개를 전송한다

```js
const fs = require('mz/fs');

app.use(async function (ctx, next) {
  const paths = await fs.readdir('docs');
  const files = await Promise.all(paths.map(path => fs.readFile(`docs/${path}`, 'utf8')));

  ctx.type = 'markdown';
  ctx.body = files.join('');
});
```

## Koa 디버깅하기

  Koa along with many of the libraries it's built with support the __DEBUG__ environment variable from [debug](https://github.com/visionmedia/debug) which provides simple conditional logging.

  For example
  to see all Koa-specific debugging information just pass `DEBUG=koa*` and upon boot you'll see the list of middleware used, among other things.

```
$ DEBUG=koa* node --harmony examples/simple
  koa:application use responseTime +0ms
  koa:application use logger +4ms
  koa:application use contentLength +0ms
  koa:application use notfound +0ms
  koa:application use response +0ms
  koa:application listen +0ms
```

  Since JavaScript does not allow defining function names at
  runtime, you can also set a middleware's name as `._name`.
  This is useful when you don't have control of a middleware's name.
  For example:

```js
const path = require('path');
const serve = require('koa-static');

const publicFiles = serve(path.join(__dirname, 'public'));
publicFiles._name = 'static /public';

app.use(publicFiles);
```

  Now, instead of just seeing "serve" when debugging, you will see:

```
  koa:application use static /public +0ms
```
