# Go语言编写Telegram评论机器人


----------------------------------------------------
2019.06.01更新

新增几个命令：
/reload 重新加载功能异常的评论区
/recover 恢复被误删的评论区
/enable 手动开启评论区

增加的内容放在dev分支

计划更新：
黑名单功能
评论删除
设置非频道订阅者的评论权限

----------------------------------------------------

----------------------------------------------------
2019.05.28更新

肝了快4天终于把第一个版本肝完了
Github[项目地址](https://github.com/Just4fan/telegram_comment_bot)

----------------------------------------------------

前几天有人问我Telegram上的频道评论机器人有没有楼中楼和通知功能，找了一下好像没找到，突然感兴趣自己写一个，看了一下Telegram提供的Bot API，觉得应该是能实现。

用Go语言写是因为只带了笔记本回家，笔记本上只装了Go的开发环境，懒得去折腾，就干脆用Go写吧。

## （一）申请机器人 ##

给 @BotFather 发送 /newbot 指令，按照指示完成机器人创建，创建成功会得到一个token，是调用Telegram API必备的，如果没有保存可以发送 /token 指令重新生成一个token，也可以发送 /revoke 指令撤销token。

## （二）环境配置 ##

关于Go开发环境的配置不再赘述，网上有很多教程。关于Telegram Bot API，官方提供了使用http(s)协议的API，这里为了方便使用Go语言封装好的API。
[API地址](https://github.com/go-telegram-bot-api/telegram-bot-api)
[API文档](https://godoc.org/gopkg.in/telegram-bot-api.v4)

在项目目录输入
```shell
go get -u github.com/go-telegram-bot-api/telegram-bot-api
```
或
```shell
go get gopkg.in/telegram-bot-api.v4
```

## （三）Bot API ##

建议在使用前先看看官方的[API文档](https://core.telegram.org/bots/api)。

### 1 Keyboards ###

我们经常可以在一些频道看到消息下方显示了按钮，这种按钮是通过在消息添加InlineKeyboards实现的，InlineKeyboards的按钮有callback(发送数据给bot)、url(跳转到链接)、switch to inline(在当前会话或选择一个新的会话输入inline query)这三种。

### 2 Update ###

Bot只有在被添加到群组、频道或者私人对话中才能接收消息。一个Update可以看做机器人接收的一个消息，当我们发送消息在群组、频道或发送消息给机器人、点击按钮产生的回调等都是一个Update。

### 3 getUpdates ###

官方提供两种方式获取更新，一种方式是使用长轮询向Telegram服务器拉取更新(返回一个Update数组)，使用这种方式需要不断的更新offset的值，offset代表获取的Update数组里第一个Update的update_id，如果offset设置不正确会获取到已经获取过的更新，另一种方式是通过设置Webhook指定一个接收Update的服务器，有新的Update时，Telegram服务器会自动将数据用POST的方式发送到指定的服务器，使用这种方式不会接收到已经接收过的更新，并且可以实时接收到更新，前提是指定的服务器需要有公网ip和https，本地开发我们可以借助ngrok把本地地址映射到公网地址上。这里采用第二种方式。

## （四）实现 ##

### 1 设置Webhook并获取更新 ###

设置Bot Token和Webhook

```go
token := "你的bot token"
bot, err := tgbotapi.NewBotAPI(token)
if err != nil {
    panic(err)
}
bot.Debug = true
//地址路径加上token可以用于验证消息推送者的身份
ret, err := bot.SetWebhook(tgbotapi.NewWebhook("https://公网地址/" + bot.Token))
if err != nil {
    panic(err)
}
log.Println(ret)
```

接收更新

```go
//返回一个 chan Update
updates := h.bot.ListenForWebhook("/" + h.bot.Token)
go http.ListenAndServe("0.0.0.0:80", nil)
for update := range updates {
    //处理Update
}
```

### 2 给频道消息加上操作按钮 ###

首先判断消息来源是不是频道

```go
if update.ChannelPost != nil {
    handleChannelMessage(update.ChannelPost)
}
```

先转发原post，然后编辑转发消息，给转发消息加上键盘。
点击按钮时打开机器人对话框，把ChatID和消息的MessageID（ChatID和MessageID共同标识一条消息）作为参数传递给机器人（用Deep-linking实现）。

![avatar](https://s2.ax1x.com/2019/05/28/VmbArF.png)

点击按钮

![avatar](https://s2.ax1x.com/2019/05/28/VmqVQf.png)

```go

config := &tgbotapi.MessageConfig{
    BaseChat: tgbotapi.BaseChat{
        ChatID:              message.Chat.ID,
        ReplyToMessageID:    message.MessageID,
        DisableNotification: true,
    },
    Text:      "暂无评论",
}
msg, err := h.bot.Send(config)


if err == nil {

    post := &models.Post{
        MessageID: message.MessageID, 
        ChatID:message.Chat.ID, 
        AreaID: msg.MessageID,
    }
    
    //生成Deep-linking参数
    param, err := utils.EncodeParam(post)
    
    if err != nil {
        log.Println(err)
        return
    }
    
    url := "http://t.me/comment_it_bot?start=" + param
    
    keyboard := utils.NewInlineKeyboardMarkup(
        tgbotapi.NewInlineKeyboardRow(
            tgbotapi.InlineKeyboardButton{
                Text: "查看详情️", 
                URL: &url,
            }))
            
    _, err = h.bot.Send(tgbotapi.NewEditMessageReplyMarkup(msg.Chat.ID, msg.MessageID, keyboard))
    h.commentRepo.InsertPost(post)
}
```

