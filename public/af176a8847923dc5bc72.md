---
title: >-
  Leafletで"Uncaught TypeError: Cannot read properties of null (reading
  '_latLngToNewLayerPoint')"が出力される
tags:
  - JavaScript
  - leaflet
private: false
updated_at: '2025-05-31T10:15:02+09:00'
id: af176a8847923dc5bc72
organization_url_name: null
slide: false
ignorePublish: false
---
## 環境
- [Vue](https://ja.vuejs.org/) 3.4.21
- [leaflet](https://leafletjs.com/) 1.9.4

## 事象
leafletでOpenStreetMapを表示し、その上にピンを描写。
その状態で下記動作を行うことでエラー発生。
- ズームイン/アウトを高速で繰り返す
- 同コンポーネントの状態が変更するような動作（例えばプルダウンを表示する等）をした直後にズームを行う

### エラー内容
```
Uncaught TypeError: Cannot read properties of null (reading '_latLngToNewLayerPoint')
    at e._animateZoom (Tooltip.js:209:23)
    at e.fire (Events.js:195:9)
    at e._animateZoom (Map.js:1701:8)
    at e.<anonymous> (Map.js:1679:9)
```

## 原因
結論としてはleafletのバグのようです。
エラー主力された`/leaflet/src/layer/Tooltip.js`の該当処理を見てみると、
`this._map._latLngToNewLayerPoint`を呼び出そうとしています。
この時に`this._map`がnullなのでエラーが発生している模様。
```js
_animateZoom: function (e) {
    var pos = this._map._latLngToNewLayerPoint(this._latlng, e.zoom, e.center);
    this._setPosition(pos);
},
```
恐らく負荷をかけて処理が遅くなる、コンポーネント再読み込みなどのタイミングで`this._map`が格納される前にズーム処理が動いているのが原因かと思います。
Issueにも同様の不具合が報告されていました。
[Fix issue with _latLngToNewLayerPoint #8574](https://github.com/Leaflet/Leaflet/pull/8574)

## 対策
上記のIssueにもありますが、問題のメソッドをオーバーライドして実行前に`this._map`が存在するかチェックすることで解消しました。
具体的には下記のコードを追記しました。
```ts
<script lang="ts" setup>
import { ref } from 'vue'
import L from 'leaflet'

// ここから
L.Tooltip.prototype._animateZoom = function (e: any) {
    if (!this._map) {
        return
    }
    let pos = this._map._latLngToNewLayerPoint(this._latlng, e.zoom, e.center)
    this._setPosition(pos)
}
L.Tooltip.prototype._updatePosition = function () {
    if (!this._map) {
        return
    }
    let pos = this._map.latLngToLayerPoint(this._latlng)
    this._setPosition(pos)
}
// ここまでを追加

const map = ref<any>()

// 地図描写処理 etc.
</script>
<template>
...
</template>
```

## 参考
- [Fix issue with _latLngToNewLayerPoint #8574](https://github.com/Leaflet/Leaflet/pull/8574)
- [Leaflet error when zoom after close popup in lightning component](https://salesforce.stackexchange.com/questions/180977/leaflet-error-when-zoom-after-close-popup-in-lightning-component)
