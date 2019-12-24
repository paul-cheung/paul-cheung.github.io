---
layout: post
title: 和大家一起学习了一把Teams上的开发
---

其实刚过完年不久，回到上海也将将一个月，感觉还是没有回到状态，哈哈哈。

这次其实，与其说是培训，不如说是一起探讨Teams上的一些开发技术和更新。

年前11月份就有朋友请我过来跟他们的小伙伴分享一下Microsoft Teams上的一些开发技术，还有一些经验。

不过说实话，经验倒真的谈不上，因为我们公司自己做的应用也是处于Beta版本，没有上App Store。

所以，只能说跟大家会儿共同探讨，一起学习吧。

## 大概的日程
#### 10:00-10:15：邀请方对我做了简单的介绍。
#### 10:15-10:45：项目带队PM介绍了一下目前的平台开发进展，及遇到的问题。
#### 10:45-11:00，针对以上问题，我做了简单的分析。
#### 11:00-12:30，直接跟大家讲干货，上手coding分析代码。

以上是简单的日程。

说到他们平台遇到的问题，其实主要还是怎么收集用户数据，再给到后台处理。

这个已经有相应的解决方案了，可以通过Task Module来做，至于用Adaptive card还是内嵌html，就看他们的实现了。

我倒是建议他们用card来做，因为UI级别的问题基本不用考虑，也就是兼容性问题。

如果用内嵌html，也能做，只不过要考虑不同平台的展示效果，可能出了力，效果还欠点儿。

在座只有五六个是对Microsoft Teams Platform有系统了解的，其他的基本上没有真正动手做过，所以权当是平台介绍，讲了一些实现方案方面的东西。哪些能实现，哪些不能。

中间也就直接从编辑manifest文件到安装tab应用，说明了开发中的一系列点。

基本上也都动手做了一遍，因为比较简单不涉及bot，所以很快大家就上手了。

至于bot方面，我跟大家讲了官方文档和开源示例项目。

下边有人讨论，如果Teams应用要上线，怎么做多语言？  

其实当时这块儿内容还在preview版本吧，不过5月20号已经release了，大家可以使用 了。用法也很简单，每种语言对应一个json文件，里边是每个字段的描述。

![microsoft-teams](../images/20190312/dot-json.png)

每个json文件里的内容，类似这样：
```json
{
  "$schema": "https://developer.microsoft.com/en-us/json-schemas/teams/v1.5/MicrosoftTeams.Localization.schema.json",
  "name.short": "应用短名",
  "name.full": "应用名称全全称",
  "description.short": "应用简短描述",
  "description.full": "应用描述全称",
  "staticTabs[0].name": "这里是Tab1名称",
  "staticTabs[1].name": "这里是Tab2名称",
  "staticTabs[2].name": "这里是Tab3名称",
  "bots[0].commandLists[0].commands[0].title": "Bot里的第一个command的title",
  "bots[0].commandLists[0].commands[0].description": "Bot里的第一个command的描述"
}
```
是不是很简单？

当然，这里有人会有疑问，commandLists[0]可以理解，因为bot可以有多个command，但是为什么是bots[0]呢？

原因就是，目前应用在manifest里只能配置一个bot，还不支持多个。所以这里的bots[0]其实是作为扩展的写法，如果以后支持多个bot在一个应用里，那么就比较容易做扩展。如此看来，巨硬真是666呀。

总结一下，中间也遇到我之前没有碰到的问题，也是现场查阅资料，跟他家一起探讨。

收获颇丰。

![microsoft-teams](../images/microsoftteams.png)