---
layout: page
title: 使用Promise来解耦异步操作与回调
description: 
date: 2017-04-24
---

# 异步操作与回调

```js
/**
* 图象识别
*/
function imgRec() {
    // 图像识别初始化

    asynGetImgData();
}

/**
* 异步获取图片内容，然后进行识别
*/
function asynGetImgData() {
    $.ajax.post({
        url: 'http://wwww.abc.com/get',
        sucess: function (data) {
            // 进行图象识别

            asynSaveImgRecResult(result);
        }
    });
}

/**
* 异步保存图象识别结果，然后显示消息到界面
*/
function asynSaveImgRecResult(result) {
    $.ajax.post({
        url: 'http://wwww.abc.com/save',
        sucess: function (msg) {
            $('#msg').text(msg); // 显示消息到界面上
        }
    });
}
```