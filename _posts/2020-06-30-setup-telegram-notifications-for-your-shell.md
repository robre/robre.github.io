---
layout: post
title: "Setup Telegram Notifications for your Shell"
categories: automation bash
---
## Setup Telegram Notifications for your Shell

Sometimes recon scripts take forever. I like to keep them running while doing something else, so I was looking for a way to get a notification when they are finished. I found that it is quite easy to set up Telegram notifications. To do so, simply follow these steps:

1. Search `@BotFather` on Telegram, and send him a `/start` message.
2. Next create a bot by sending the `/newbot` command.
3. Follow the Botfathers instructions and give your bot a name
4. You will receive an API token. Save it.
5. Open a shell
6. Run `telegram_api_token="APITOKEN"` with your API token
7. In Telegram, search for the Name that you gave your Bot, and send it a `/start` message.
8. In your shell, run `curl "https://api.telegram.org/bot$telegram_api_token/getUpdates" | jq '.result[0].message.chat.id'` to get your chat_id
9. add the following to your `.zshrc`, or `.bashrc`
```bash
tnotify(){
    message=$1
    token="YOURAPITOKEN"
    chatid="YOURCHATID"
    curl -s -X POST https://api.telegram.org/bot$token/sendMessage -d chat_id=$chatid -d text="$message"
}
```

Now you can run `tnotify "hi"` in your shell to get a notification!
