---
title: 流量控制设计
keywords: ["flow-control"]
description:  Apache ShenYu 流量控制设计
---

`Apache ShenYu`网关通过插件、选择器和规则完成流量控制。相关数据结构可以参考之前的 [ShenYu Admin数据结构](./database-design) 。

## 插件

* 在`shenyu-admin`后台，每个插件都用`handle`（`json`格式）字段来表示不同的处理，而插件处理就是用来管理编辑`json`里面的自定义处理字段。
* 该功能主要是用来支持插件处理模板化配置的。


## 选择器和规则

选择器和规则是 `Apache ShenYu` 网关中最灵魂的设计。掌握好它，你可以对任何流量进行管理。

 一个插件有多个选择器，一个选择器对应多种规则。选择器相当于是对流量的一级筛选，规则就是最终的筛选。
对一个插件而言，我们希望根据我们的配置，达到满足条件的流量，插件才会被执行。
 选择器和规则就是为了让流量在满足特定的条件下，才去执行我们想要的，这种规则首先要明白。

插件、选择器和规则执行逻辑如下，当流量进入到`Apache ShenYu`网关之后，会先判断是否有对应的插件，该插件是否开启；然后判断流量是否匹配该插件的选择器；然后再判断流量是否匹配该选择器的规则。如果请求流量能满足匹配条件才会执行该插件，否则插件不会被执行，处理下一个。`Apache ShenYu`网关就是这样通过层层筛选完成流量控制。

<p align="center">
    <img src="/img/shenyu/plugin/plugin-chain-execute.png" width="40%" height="30%" />
</p>

## 流量筛选

<p align="center">
    <img src="/img/shenyu/design/flow-condition.png" width="30%" height="30%" />|
</p>

流量筛选，是选择器和规则的灵魂，对应为选择器与规则里面的匹配条件(conditions)，根据不同的流量筛选规则，我们可以处理各种复杂的场景。流量筛选可以从`Header`, `URI`,  `Query`, `Cookie` 等等Http请求获取数据，

然后可以采用 `Match`，`=`，`SpEL`，`Regex`，`Groovy`，`Exclude`等匹配方式，匹配出你所预想的数据。多组匹配添加可以使用And/Or的匹配策略。


具体的介绍与使用请看: [选择器与规则管理](../user-guide/admin-usage/selector-and-rule) 。


```mermaid
stateDiagram-v2
    state "插件1..n" as p1
    state pc1 <<choice>>
    state "选择器1..n" as s1
    state sc1 <<choice>>
    state "规则1..n" as r1
    state rc1 <<choice>>
    state "执行规则" as rr1
    state rrc1 <<choice>>

    [*] --> p1

    state p1 {
        [*] --> pc1
        pc1 --> [*] : 插件未开启<br/>继续执行下一个插件

        state s1 {
            [*] --> sc1
            sc1 --> [*] : 选择器未匹配<br/>继续匹配下一个选择器

            state r1 {
                [*] --> rc1
                rc1 --> [*] : 规则未匹配<br/>继续匹配下一个规则

                state rr1 {
                    [*] --> rrc1
                    rrc1 --> [*] : 继续执行下一个插件
                }
                rc1 --> rr1 : 规则已匹配<br/>开始执行规则
            }
            sc1 --> r1 : 选择器已匹配<br/>开始匹配规则
            note left of r1 : 当选择器类型为《全流量》时<br/>该选择器和其下规则<br/>的条件均无效<br/>取末尾规则执行
        }
        pc1 --> s1 : 插件已开启<br/>开始匹配选择器
        note left of s1 : 在一个选择器里，至多只会匹配&执行一条规则
    }
    note left of p1 : 在一个插件里，至多只会匹配到一个选择器
```
