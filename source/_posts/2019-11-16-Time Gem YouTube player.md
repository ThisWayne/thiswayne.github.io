---
title: Time Gem YouTube player
date: 2019-11-16 16:59:07
urlname: time-gem-youtube-player
tags:
  - Youtube IFrame API
  - Youtube Player
  - Drum tool
  - Time Gem YouTube player
---

我需要這樣的一個網頁的情況是，我本身在學習爵士鼓，有些時候老師會教一首歌怎麼打，但一開始肢體協調性以及對歌不夠熟悉，沒有辦法以歌曲的原速度跟著打鼓，所以會先用慢一點的速度，通常播放的情況是用手機的YouTube APP，而YouTube APP的速度調整只能以0.25倍為單位，我常用的是0.75倍或是1倍，但0.75倍到1倍有時候差太多，通常來說練習會以5~10個BPM慢慢往上加，所以打算寫一個靜態網頁，功能是貼上YouTube網址然後有幾個大按鈕方便手機使用，可以一點一點的加速、可以設定AB段循環。

<!-- more -->

## 已知可以控制瀏覽器影片播放器的速度

用電腦的瀏覽器看YouTube即使調到2倍速有時候還是覺得速度很慢，所以會直接在console寫個JavaScript程式調到想要的速度，所以我應該可以在自己的網頁上用一個iframe把YouTube的影片嵌入在裡面，然後從自己的網頁上看怎麼控制嵌入在裡面的YouTube影片？

## YouTube官方的嵌入影片播放器API

查了一下，YouTube官方本身就有寫一個如果要嵌入YouTube影片可以用的library

[YouTube Player API Reference for iframe Embeds](https://developers.google.com/youtube/iframe_api_reference)
[YouTube Embedded Players and Player Parameters](https://developers.google.com/youtube/player_parameters)

## YouTube在桌面版瀏覽器上播放可以客製化播放速度

弄到一半發現原來電腦版瀏覽器上播放速度可以0~2倍速度客製化，之前都直接按快捷鍵調整沒發現呵呵，但是手機板的APP還是沒有辦法小刻度的調，用手機的瀏覽器然後選擇電腦版的網頁就可以，不過用手機去操作桌面版很難用就是了。

## 如果YouTube的JavaScript library可以調iframe裡面的YouTube播放器速度，我應該也可以自己寫個JavaScript調iframe裡面的播放器速度？

本來一開始因為自己的程式有bug，透過YouTube的library一直控制不到速度，想說乾脆自己從iframe裡面DOM裡面抓video控制影片速度，順便學到了因為會有資安問題所以不能這樣硬幹，網頁與內嵌iframe網頁的溝通方法，要靠Window.postMessage，簡單來說就是網頁透過postMessage把資料傳給內嵌網頁，內嵌網頁透過message event接收資料，用chrome的話可以用monitorEvents(window, "message")在console來看兩邊溝通的log，以YouTube的情況來看，內嵌網頁會把可以呼叫的interface資訊提供給外面的網頁呼叫，只有他有定義的interface呼叫了有用。

## 成品Demo

只是個小網站，所以沒有用上任何JS框架，也沒有用jQuery，直接用原生DOM API，CSS要針對HTML對應巢狀有點小亂所以後來用Sass寫，整體的layout沒有太複雜的邏輯所以沒有用上media query來做RWD，畫面元件設定最小寬高不要讓手機版手指不好按就好，整體都靠flex做layout。

有一些比較麻煩的地方反而是YouTube本身給的API不方便使用，要自己寫一些hack來解，像是沒有單獨的API可以直接告訴撥放器撥到哪裡就好，只有讀取並播放影片同時給起始與結束時間的API，但是呼叫那個API影片就又重新下載很浪費網路流量；像是有一個API可以load進影片，但是剛呼叫完就呼叫詢問影片長度的API可能因為還沒有載到影片meta data還沒有準備好所以呼叫失敗，只好自己寫一個setInterval去更新影片長度，有點多餘。

[Time Gem YouTube player](https://thiswayne.github.io/TimeGemYoutubePlayer/)
