---
layout: post
title: C#代码中如何解析Bot里的事件
---

![microsoft-teams](../images/microsoftteams.png)

在之前的一篇文章里简单介绍了一下Bot事件，大概有哪些，可以在哪些场景下使用，使用范围是什么样的。

今天我们就来看一个实例，使用ASP.NET MVC框架写一个Bot，在Teams中安装Bot后，跟本项目中的代码进行消息交互。

如果有还知不道如何玩的童鞋，可以参考我的其他文章。这里就假设大家都可以本地调试起来项目了，我们直接看Bot的后台代码，进行解释。

不多废话，首先看看Bot的endpoint，如下：
```csharp
using System;
using System.Net;
using System.Net.Http;
using System.Threading.Tasks;
using System.Web.Http;
using Microsoft.Bot.Connector;
using Microsoft.Bot.Connector.Teams;
using Microsoft.Bot.Connector.Teams.Models;

namespace Microsoft.Teams.Samples.TaskModule.Web.Controllers
{
    [BotAuthentication]
    public class MessagesController : ApiController
    {
        [HttpPost]
        public async Task<HttpResponseMessage> Post([FromBody] Activity activity)
        {
            using (var connector = new ConnectorClient(new Uri(activity.ServiceUrl)))
            {
                if (activity.IsComposeExtensionQuery())
                {
                    // 这里是处理MessageExtension的逻辑，暂且不提
                }
                else
                {
                    // 这里是我们处理Bot event的逻辑，是我们这篇博客的重点，下一步我们就着重看看Bot.HandleActivity方法。
                    var result = await Bot.HandleActivity(connector, activity);
                    return result == null ?
                    new HttpResponseMessage(HttpStatusCode.OK) :
                    Request.CreateResponse(HttpStatusCode.OK, result);
                }
            }
        }
    }
}

```

以上是endpoint的主要代码，没啥可说的，老司机们一眼就明白。

再看看Bot类里的逻辑：

```c#
public class Bot
    {
        public static async Task<object> HandleActivity(ConnectorClient connector, Activity activity)
        {
            switch (activity.Type)
            {
                // Activity的类型为conversationUpdate
                case "conversationUpdate":
                    {
                        // 获取当前操作的用户;
                        ChannelAccount operatedMember = null;
                        // 这里偷个懒，使用dynamic解析数据，如果真正到产品化的项目中，建议使用正规的model解析方式。
                        var eventType = (string)((dynamic)activity.ChannelData)["eventType"];
                        switch (eventType)
                        {
                            // 事件类型为添加团队成员
                            case "teamMemberAdded":
                                {
                                    operatedMember = activity.MembersAdded[0];
                                    await SendAddOrRemoveTeamMemberMsg(connector, activity, operatedMember, true);
                                    break;
                                }
                            // 事件类型为移除团队成员
                            case "teamMemberRemoved":
                                {
                                    operatedMember = activity.MembersRemoved[0];
                                    await SendAddOrRemoveTeamMemberMsg(connector, activity, operatedMember, false);
                                    break;
                                }
                            // 事件类型为重命名团队
                            case "teamRenamed":
                                {
                                    await SendRenameTeamMsg(connector, activity);
                                    break;
                                }
                            // 事件类型为创建频道
                            case "channelCreated":
                                {
                                    await SendChannelCreatedMsg(connector, activity);
                                    break;
                                }
                            // 事件类型为重命名频道
                            case "channelRenamed":
                                {
                                    await SendChannelRenamedMsg(connector, activity);
                                    break;
                                }
                            default:
                                break;
                        }
                        break;
                    }
                // Activity的类型为messageReaction，也就是点赞/取消赞的逻辑
                case "messageReaction":
                    {
                        // 数据结构上，这里在activity的对象中有ReactionsAdded和ReactionsRemoved两个数组。
                        // 这里用like判断类型
                        var reactionsAdded = activity.ReactionsAdded?.Where(r => r.Type == "like").ToList();
                        var reactionRemoved = activity.ReactionsRemoved?.Where(r => r.Type == "like").ToList();

                        // 以下就是根据点赞还是取消赞返回响应消息。
                        if (reactionsAdded != null && reactionsAdded.Count > 0)
                        {
                            await SendReactiondMsg(connector, activity, reactionsAdded, true);
                        }
                        if (reactionRemoved != null && reactionRemoved.Count > 0)
                        {
                            await SendReactiondMsg(connector, activity, reactionRemoved, false);
                        }
                        break;
                    }
                default:
                    {
                        // Nothing to do;
                        break;
                    }
            }
            return null;
        }
    }
```

以上用到里几个Bot类中的私有方法，以下给出解释，首先是SendAddOrRemoveTeamMemberMsg
```c#
        public static async Task SendAddOrRemoveTeamMemberMsg(ConnectorClient connector, Activity activity, ChannelAccount operatedMember, bool isAdd)
        {
            // 欢迎消息还是欢送消息
            var replyMsg = isAdd ? "Welcome you come here" : "Goodbye~~~";

            // 获取聊天组里的成员，可以进行缓存噢
            var members = (await connector.Conversations.GetConversationMembersAsync(activity.Conversation.Id)).ToList();
            var memberName = members.Find(m => m.Id == operatedMember.Id)?.Name;

            // 构造返回的Activity
            var replyActivity = new Activity();
            replyActivity.Text = replyMsg;
            if (!string.IsNullOrEmpty(memberName))
            {
                // 构造At成员的消息文本
                replyActivity.AddMentionToText(operatedMember, MentionTextLocation.PrependText, $"@{memberName}");
            }

            replyActivity.Type = "message";
            replyActivity.Conversation = new ConversationAccount() { Id = activity.Conversation.Id };

            await connector.Conversations.SendToConversationAsync(replyActivity);
        }
```
其中，SendRenameTeamMsg，SendChannelCreatedMsg，SendChannelRenamedMsg方法这里就不作分析了，大家可以根据上边的这个方法自行补充。

这里主要看点赞的这个方法，其实方法体也都差不多。
```c#
        public static async Task SendReactiondMsg(ConnectorClient connector, Activity activity, IList<MessageReaction> messageReactions, bool isAdd)
        {
            var msg = string.Empty;
            // 点赞
            if (isAdd)
            {
                msg = $"Thanks for your like";
            }
            // 取消赞
            else
            {
                msg = $"Thanks for your unlike";
            }

            // 谁操作的
            var members = (await connector.Conversations.GetConversationMembersAsync(activity.Conversation.Id)).ToList();
            var memberName = members.Find(m => m.Id == activity.From.Id)?.Name;

            var replyActivity = new Activity();
            replyActivity.Text = msg;
            replyActivity.Type = "message";
            if (!string.IsNullOrEmpty(memberName))
            {
                replyActivity.AddMentionToText(new ChannelAccount() { Id = activity.From.Id }, MentionTextLocation.PrependText, $"@{memberName}");
            }
            replyActivity.Conversation = new ConversationAccount() { Id = activity.Conversation.Id };

            await connector.Conversations.SendToConversationAsync(replyActivity);
        }
```

有了以上的逻辑，你的Bot就可以回复团队成员的点赞和取消赞了。

看看效果：
![welcome-reaction](../images/20190210/welcome-reaction.png)

还有其他一系列事件都可以根据我们自己的业务场景进行扩充。

今天就先跟大家分享到这里了，后续如果有什么好玩的，我会再跟大家share的~~~

冲🦆🦆🦆🦆🦆🦆！！！