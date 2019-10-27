---
layout: post
title: "Axios와 Retry"
author: "genie-youn"
categories: journal
tags: [axios]
image: alberto-barrera-G3GL5lJygz8-unsplash.jpg
---
<center><small><a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px" href="https://unsplash.com/@abarro?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Alberto Barrera"><span style="display:inline-block;padding:2px 3px"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-2px;fill:white" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M10 9V0h12v9H10zm12 5h10v18H0V14h10v9h12v-9z"></path></svg></span><span style="display:inline-block;padding:2px 3px">Alberto Barrera</span></a></small></center>

# Axios와 Retry

### 들어가며
axios는 실패시 체인의 중간단계에서나 설정을 통한 retry를 제공하지 않는다. 사용자가 직접 구현해 주어야 하는데, 방법은 크게 다음과 같다.

### Interceptor 활용
실패시 인터셉터로 잡아 재시도 하는 방법이다. axios 개발팀도 이 방법을 권하고 있다.

```javascript
axios.interceptors.response.use(null, (error) => {
  if (error.config && error.response && error.response.status === 401) {
    return updateToken().then((token) => {
      error.config.headers.xxxx <= set the token
      return axios.request(config);
    });
  }

  return Promise.reject(error);
});
```

> 참고 https://github.com/axios/axios/issues/934#partial-timeline

### axios-retry 라이브러리 사용
또 다른 방법은 서드파티 라이브러리인 `axios-retry`를 사용하는 방법이다.

```shell
$ npm install axios-retry
```

```javascript
// ES6
import axiosRetry from 'axios-retry';

axiosRetry(axios, { retries: 3 });

axios.get('http://example.com/test') // The first request fails and the second returns 'ok'
  .then(result => {
    result.data; // 'ok'
  });

// Exponential back-off retry delay between requests
axiosRetry(axios, { retryDelay: axiosRetry.exponentialDelay});

// Custom retry delay
axiosRetry(axios, { retryDelay: (retryCount) => {
  return retryCount * 1000;
}});

// Works with custom axios instances
const client = axios.create({ baseURL: 'http://example.com' });
axiosRetry(client, { retries: 3 });

client.get('/test') // The first request fails and the second returns 'ok'
  .then(result => {
    result.data; // 'ok'
  });

// Allows request-specific configuration
client
  .get('/test', {
    'axios-retry': {
      retries: 0
    }
  })
  .catch(error => { // The first request fails
    error !== undefined
  });
```

사용량도 꽤나 많고, back-off나 재시도 횟수 설정등이 이미 구현되어 있어 편하게 사용할 수 있다는 장점을 갖지만, axios에서 개발한 오피셜한 라이브러리는 아니기때문에 향후 꾸준히 업데이트 될지는 의문이다.
