---
layout: post
title: "크롬 개발자도구에서 Remote Device를 못 찾을 때"
author: "genie-youn"
categories: journal
tags: [chrome, debugging]
image: ian-robinson-EzGRCDqLPaY-unsplash.jpg
---
<center><small><a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px" href="https://unsplash.com/@ianrobinson?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Ian Robinson"><span style="display:inline-block;padding:2px 3px"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-2px;fill:white" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M10 9V0h12v9H10zm12 5h10v18H0V14h10v9h12v-9z"></path></svg></span><span style="display:inline-block;padding:2px 3px">Ian Robinson</span></a></small></center>


인앱 브라우저에서 동작하는 스크립트를 디버깅하기 위해 늘 하던대로 개발자도구를 켜고, USB 디버깅을 켠다음, USB를 연결해서 크롬의 Remote Device 탭을 열었는데 왠걸 디바이스를 찾을 수 없는 일이 발생했다.
> 자세한 방법은 [여기](https://developers.google.com/web/tools/chrome-devtools/remote-debugging?utm_campaign=2016q3&utm_medium=redirect&utm_source=dcc)

<br>
### 삽질
승인된 기기를 전부 지워보기도 하고, 옆자리 동료 맥에도 꽂아보고 케이블도 바꿔봤지만 읽힐 생각은 전혀 없던 찰나 스택오버플로우에서 Android 디버그 브리지(adb)를 받아서 돌려보라는 글을 하나 보았다.

이 [링크](https://developer.android.com/studio/releases/platform-tools) 에 들어가서 `./adb devices` 를 돌려보면 콘솔에 연결되있던 디바이스를 찾을 수 있는것을 확인할 수 있다. 그러면 크롬에도 연결된 디바이스가 떠 있는 것을 볼 수 있다.

> adb에 대한 자세한 설명은 [여기](https://developer.android.com/studio/command-line/adb?gclid=Cj0KCQjwuNbsBRC-ARIsAAzITuedSIz0vnnWTeIax-fiI4dQw6x0HU7ZgZeBB-Po51oSD-LUNVCk7a8aAvV4EALw_wcB) 를 참고한다.

<br>
### 결론
원인은 모르겠지만.. 늘 하던게 안되서 두시간은 해맸던 것 같다. ㅠ.ㅠ
