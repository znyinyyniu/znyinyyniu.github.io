---
layout: post
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
如上代码所示，asynGetImgData方法中，耦合了图象识别的代码，耦合了asynSaveImgRecResult方法的调用。

要是能写出如下的流水帐式的代码，那该有多好啊。
```js
/**
* 图象识别
*/
function imgRec() {
    // 图像识别初始化

    var imgData=asynGetImgData();

    // 进行图像识别

    asynSaveImgRecResult(result);

    $('#msg').text(msg); // 显示消息到界面上
}

/**
* 异步获取图片内容
*/
function asynGetImgData() {
    $.ajax.post({
        url: 'http://wwww.abc.com/get',
        sucess: function (data) {
        }
    });
}

/**
* 异步保存图象识别结果
*/
function asynSaveImgRecResult(result) {
    $.ajax.post({
        url: 'http://wwww.abc.com/save',
        sucess: function (msg) {
        }
    });
}
```
由于asynGetImgData、asynSaveImgRecResult方法都是异步操作，上述代码是不能正常工作的。

# Promise

Promise就是用来解决这个问题的，一个promise代表了一个异步操作的结果（出现了异常、成功返回了结果），因此我们就能以流水帐式的编码风格来获取异步操作的返回值，或者是处理异步操作的异常。 

上述的代码，使用Promise后会是这个样子：
```js
/**
* 图象识别
*/
function imgRec() {
    // 图像识别初始化

    var promise=asynGetImgData();
    promise.then(function(imgData){
        // 进行图像识别
        var promise1=asynSaveImgRecResult(result);
        promise1.then(function(msg){
            $('#msg').text(msg); // 显示消息到界面上
        })
        .catch(function(err){
            console.log('保存图像识别的结果出错！');
        });
    }).catch(function(err){
        console.log('获取图像数据出错！');
    }); 
}

/**
* 异步获取图片内容
*/
function asynGetImgData() {
    var promise=new Promise(function(resolve,reject){
        $.ajax.post({
            url: 'http://wwww.abc.com/get',
            sucess: function (imgData) {
                resolve(imgData);
            }
            error: function(err){
                reject(err);
            }
        });
    });
    return promise;
}

/**
* 异步保存图象识别结果
*/
function asynSaveImgRecResult(result) {
    var promise=new Promise(function(resolve,reject){
        $.ajax.post({
            url: 'http://wwww.abc.com/save',
            sucess: function (msg) {
                resolve(msg);
            }
            error: function(err){
                reject(err);
            }
        });
    });
    return promise;
}
```