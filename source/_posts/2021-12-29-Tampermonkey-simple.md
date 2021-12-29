---
title: 油猴入门
date: 2021-12-29
tags: TamperMonkey
---

# 1、入门，一个简单的模版
```js
// ==UserScript==
// @name         清理文档库前后行
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  try to take over the world!
// @author       palexu
// @match        https://metis.xxx.com/**
// @icon         
// @grant        none
// ==/UserScript==

(function () {
    const slct = '#watermark > div.s-document-edit > div.editor-wrapper > div > div';

    function run() {
        let allElements, thisElement;
        allElements = document.querySelector(slct).childNodes;

        console.log(allElements);

        for (let i = 0; i < allElements.length; i++) {
            thisElement = allElements[i];
            // 使用 thisElement
            console.log(i, thisElement);
            if ("\n" === thisElement.outerText) {
                console.log("found");
                thisElement.remove();
            } else {
                break;
            }
        }
    }


    let cleanBtn = document.createElement('button');
    cleanBtn.className = `ant-btn ant-btn-warning`;
    cleanBtn.style = "margin-left: 15px;";
    cleanBtn.innerHTML="<span>清理空行</span>";
    cleanBtn.onclick = run;
    document
        .querySelector('#watermark > div.s-document-edit > div.s-document-edit-header > div.s-document-edit-submit')
        .append(cleanBtn);


})();
```

