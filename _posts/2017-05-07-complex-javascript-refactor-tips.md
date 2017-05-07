---
layout: post
title: 复杂javascript代码重构技巧
description: 
date: 2017-05-07
---

# 背景

项目中常会有一些超级复杂的javascript文件，可能有几万行代码。这些文件大多采用面向过程的方式来组织，经过日积月累的堆叠，其中的代码变得十分难以阅读和维护。因此，对这些复杂的javascript代码进行重构就显得十分必要。

# 面向对象，遵从类的单一职责原则

采用面向对象的思想来组织复杂的代码，是一个很好的办法。将大型的代码文件按功能职责进行划分，第一个职责交给一个类去实现，确保各个类职责清晰不混淆，这样就能将复杂的代码简化，从而更加易于阅读和维护。TypeScript是javascript的超集，采用TypeScript来实现面向对象，会比javascript更能提高开发效率，代码的规范化方面也更有优势。

1. MathML4Img。
2. DomCtrl4Img中的readAndUploadImgs方法。
3. DomCtrl4Height、HeightUtil、Utils

# 采用工厂设计模式，来避免大量重复的条件分支

在一个类的诸多方法中，都会重复的使用相同的条件分支判断。此时，可以设计一个子类，重载父类中的方法，实现不同的条件分支中的逻辑。然后设计一个工厂类，它能够根据不同的条件分支产生不同的类实例。这样条件分支的判断就集中到了工厂类中，减少了代码的重复，也便于以后修改。由于类中各方法都去掉了条件分支的判断，代码逻辑更加清晰，便于阅读。由于采用了继承重载的设计方法，所以更容易扩展出不同功能的子类。

1. DomCtrl4PaperName、DomCtrl4PaperName4HandScore、DomCtrl4PaperNameFactory。
2. DomCtrl4Height、DomCtrl4Height4Fill、DomCtrl4HeightFactory。

# 采用高内聚原则来组织代码

功能职责相同的代码分散在不同的文件中，这是一种很不好的习惯，这样会导致在修改功能或Bug时产生遗忘，最终导致功能或Bug修改不完全。可以将这些代码都内聚到一个类中，进行统一组织与设计，这样代码就易于维护，也易于阅读了。

1. DomCtrl4Img中的appendMathML方法。

# 采用Promise来解耦异步操作与回调

详见[使用Promise来解耦异步操作与回调](/2017/04/24/use-promise-resolve-asyn-and-callback.html)

1. InPageService。

# 其他

1. TypeScript中如何使用html页面中定义的全局变量？

    通过window对象来使用，因为，html页面中定义的全局变量都属于window对象。

    ```ts
    declare var window: any;
    let path = window.basePath+'';
    ```

2. 使用模板字符串来格式化字符串。

    ```ts
    static calcPxHeight4TopicTitle(html: string, width: number=150, lineHeight: number=10): number {
        let result = 0;

        let tmepHtml = `
        <div id="tempDivArea" style="display:block;white-space:pre-wrap;font-weight: bold;padding: 0 2mm;box-sizing: border-box;
            width:${width}mm;line-height:${lineHeight}mm;">
            ${html}
        </div>
        `
        $(document.body).append(tmepHtml);
        let tempDiv = $("#tempDivArea");
        result = tempDiv.height();
        tempDiv.remove();
        return result;
    }
    ```

3. 使用类型别名来表明参数的真正类型。

    ```ts
    export type jqobj = any;

    /**
     * 可拖动。
     * 
     * @private
     * @static
     * @param {jqobj} selector 
     * 
     * @memberof DomCtrl4Img
     */
    private static draggable(selector: jqobj) {
        selector.draggable({
            onDrag: DomCtrl4Img.limitInParentArea
        })
    }
    ```

4. 如何在html片段中将事件绑定到TypeScript所编写的类的方法上。

    通过window对象，因为，html片段中无法也TypeScript模块直接交互，要借助window对象。

    ```ts
    declare var window: any;

    private static onMouseEnter(ele: jqobj) {
        /* TODO
        if (window.parent.DescCtrl.canEdit()) {
        }
        */
        let topicId = ele.attr("opid");
        let type = ele.attr("category");
        let html = `
        <div id="control" class="btn_mimax" style="position:absolute;z-index: 1;right: 0mm;top:0mm;">
            <a class="btn_max_sm" style="cursor: pointer" onclick="window.domCtrl4ChoiceArea.addQuestions('${type}','${topicId}')" title="添加题目">添加题目</a>
            <a class="btn_del_sm" style="cursor: pointer" onclick="window.domCtrl4ChoiceArea.deleteQuestions('${type}','${topicId}')" title="删除">删除</a>
        </div>
        `;
        ele.append(html);
    }

    window.domCtrl4ChoiceArea = DomCtrl4ChoiceArea;
    ```

