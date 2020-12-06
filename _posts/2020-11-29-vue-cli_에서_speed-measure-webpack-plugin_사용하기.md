---
layout: post
title: "vue-cli에서 speed-measure-webpack-plugin 사용하기"
author: "genie-youn"
categories: journal
tags: [webpack, vue-cli, speed-measure-webpack-plugin]
image: IMG2127.JPG
---

## 들어가며
성능 트러블슈팅의 첫 걸음은 성능측정에 있다.

webpack의 소요 시간을 측정해주는 플러그인인 `speed-measure-webpack-plugin`을 `vue-cli`에서 사용하는 방법을 기록해두려 한다.

## speed-measure-webpack-plugin
vue-cli에 적용하기 위해선, 우선 이 플러그인이 어떻게 동작하는지 알아둘 필요가 있다.
[공식 도큐먼트](https://github.com/stephencookdev/speed-measure-webpack-plugin#usage)에 따르면

다음과 같은 webpack config 를

<small>webpack.config.js</small>
```javascript
const webpackConfig = {
  plugins: [
    new MyPlugin(),
    new MyOtherPlugin()
  ]
}
```
다음과 같이 변경하여 사용할 수 있다고 한다.

```javascript
const SpeedMeasurePlugin = require("speed-measure-webpack-plugin");

const smp = new SpeedMeasurePlugin();

const webpackConfig = smp.wrap({
  plugins: [
    new MyPlugin(),
    new MyOtherPlugin()
  ]
});
```

그럼 이렇게 각 loader와 plugin이 얼마나 많은 시간을 소요했는지 시간을 측정해준다.


![image1](https://github.com/stephencookdev/speed-measure-webpack-plugin/blob/master/preview.png?raw=true)

`SpeedMeasurePlugin`의 `wrap` 메소드에 webpack config 객체를 넘겨주면 시작과 종료시에 무언가를 하겠구나.. 하고 예측할 수 있다. 이 부분을 조금 더 살펴보면

<small>[speed-measure-webpack-plugin/index.js](https://github.com/stephencookdev/speed-measure-webpack-plugin/blob/fa1d273843801c244bbdda45ce98684f9153595c/index.js#L35-L69)<small>
```javascript
wrap(config) {
    if (this.options.disable) return config;
    if (Array.isArray(config)) return config.map(this.wrap);
    if (typeof config === "function")
      return (...args) => this.wrap(config(...args));

    config.plugins = (config.plugins || []).map(plugin => {
      const pluginName =
        Object.keys(this.options.pluginNames || {}).find(
          pluginName => plugin === this.options.pluginNames[pluginName]
        ) ||
        (plugin.constructor && plugin.constructor.name) ||
        "(unable to deduce plugin name)";
      return new WrappedPlugin(plugin, pluginName, this);
    });

    if (config.optimization && config.optimization.minimizer) {
      config.optimization.minimizer = config.optimization.minimizer.map(
        plugin => {
          return new WrappedPlugin(plugin, plugin.constructor.name, this);
        }
      );
    }

    if (config.module && this.options.granularLoaderData) {
      config.module = prependLoader(config.module);
    }

    if (!this.smpPluginAdded) {
      config.plugins = config.plugins.concat(this);
      this.smpPluginAdded = true;
    }

    return config;
  }
```

파라미터로 전달받은 `config` 객체의 `plugins` 와 `module`, `optimization`를 wrapped한 객체로 변경하고 있고, 아마 이 Wrapper가 시작과 종료시에 메트릭을 수집하겠구나 하고 예측할 수 있다.

webpack config 객체를 직접 수정한다는 점에 유의하면서 이 플러그인은 `vue-cli`에 적용해보자.


## vue.config.js
다음과 같은 vue-cli의 설정파일이 있다고 가정해보자.

vue.config.js
```javascript
module.exports = {
  configureWebpack: {
    plugins: [
      new MyAwesomeWebpackPlugin()
    ]
  },
  chainWebpack: config => {
    // GraphQL Loader
    config.module
      .rule('graphql')
      .test(/\.graphql$/)
      .use('graphql-tag/loader')
        .loader('graphql-tag/loader')
        .end()
      // Add another loader
      .use('other-loader')
        .loader('other-loader')
        .end()

    config
      .plugin('html')
      .tap(args => {
          return [/* new args to pass to html-webpack-plugin's constructor */]
      })
  }
}
```

위와 같이 vue.config.js에서 webpack을 설정할 수 있는 부분은 `configureWebpack`와 `chainWebpack` 두 부분이다. 두 옵션에 자세한 설명은 [레퍼런스](https://cli.vuejs.org/guide/webpack.html#working-with-webpack)를 참고한다.

각 부분에서 `speed-measure-webpack-plugin`를 적용하는 방법을 찾아보자.

## configureWebpack
`configureWebpack`에 `speed-measure-webpack-plugin`를 적용하는 방법은 간단하다. 몽땅 싸주면 된다.

vue.config.js
```javascript
const SpeedMeasurePlugin = require("speed-measure-webpack-plugin");
const smp = new SpeedMeasurePlugin();

module.exports = {
  configureWebpack: smp.wrap({
    plugins: [
      new MyAwesomeWebpackPlugin()
    ]
  }),
  // 생략
}
```

`npm run build`를 실행시켜 보면 각 loader와 plugins의 소요 시간이 나오는걸 확인할 수 있다. `configureWebpack`은 [`webpack-merge`](https://github.com/survivejs/webpack-merge) 를 통해 적용되기 때문에 wrapped 된 플러그인이 정상적으로 전달되어 실행되는것을 확인할 수 있다.

## chainWebpack

> ⚠️ chainWebpack 에서 해당 플러그인은 `preload`, `prefetch` 플러그인과 충돌을 일으켜 `TypeError: Cannot read property 'tap' of undefined` 에러가 발생한다. `preload`, `prefetch` 플러그인을 사용하지 않는 환경에서만 사용 가능하다.

`chainWebpack` 에서 `speed-measure-webpack-plugin`를 적용하는건 조금 복잡하다.

<small>vue.config.js</small>
```javascript
module.exports = {
  // 생략
  chainWebpack: config => {
    // preload, prefetch 플러그인을 사용하지 않는 환경에서만 사용할 수 있다.
    config.plugins.delete('preload');
    config.plugins.delete('prefetch');
    // GraphQL Loader
    config.module
      .rule('graphql')
      .test(/\.graphql$/)
      .use('graphql-tag/loader')
        .loader('graphql-tag/loader')
        .end()
      // Add another loader
      .use('other-loader')
        .loader('other-loader')
        .end()

    config
      .plugin('html')
      .tap(args => {
          return [/* new args to pass to html-webpack-plugin's constructor */]
      })

    const wrappedConfig = smp.wrap(config.toConfig());
    config.toConfig = () => wrappedConfig;
  }
}
```

스택오버플로우등을 검색해보면 [`webpack-chain`](https://github.com/neutrinojs/webpack-chain) 환경에서 `smp.wrap(config.toConfig())` 를 사용하면 된다고 하지만

`toConfig()` 된 객체를 `smp.wrap` 으로 수정해봤자, 원본인 `config` 가 수정되진 않는다. `toConfig` 메소드를 오버라이드하여 wrapped 된 config 객체를 반환하도록 하자.

## vue-cli 직접 수정하기
마지막으로 `node_modules`의 vue-cli 코드를 직접 수정하여 webpack을 실제로 호출하기 직전에 config 객체를 `smp.wrap` 으로 수정하는것이다.

vue-cli가 사용자의 config 설정이 끝난 후 추가하는 loader, plugins의 메트릭을 측정할때 필요하다. (`--report` 옵션을 주었을 경우 추가되는 `webpack-bundle-analyzer` 같은 플러그인)

`node_modules/@vue/cli-service/libs/commands/build/index.js` 의 [이 부분](https://github.com/vuejs/vue-cli/blob/c64afc3c2a8854aae30fbfb85e92c0bb8b07bad7/packages/%40vue/cli-service/lib/commands/build/index.js#L198)에 다음과 같이 수정해주면 된다.

> ⚠️ 마찬가지로 preload, prefetch 플러그인을 사용하지 않는 환경이어야 한다.

<small>index.js</small>
```javascript
if (args.clean) {
  await fs.remove(targetDir)
}

const SpeedMeasurePlugin = require("speed-measure-webpack-plugin");
const smp = new SpeedMeasurePlugin();

webpackConfig = smp.wrap(webpackConfig);

return new Promise((resolve, reject) => {
  webpack(webpackConfig, (err, stats) => {
```
