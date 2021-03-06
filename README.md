# Slack Dev Tools

These are some tools meant to be used with Slack's Incoming and Outgoing Web Hooks.

It is slowly evolving into a tool that lets you set permissions on actions based on users or teams and allows for one server to manage multiple team's incoming / outoging webhooks.

Completed modules are:

- *cli* : A command line interface with whitelists, blacklists, output filters and command aliases.
- *devtools* : A module letting you get the user ids, channel ids, team ids, service ids that are created by Slack servers
- *twitter* : Let's you tweet from Slack and use bitly to shorten your URLs
- *xteam* : A cross team communication module, letting you share channels between teams
- *mandrillnotify* : A service that lets you send your Mandrill Webhooks to the server and get displayed in a Slack channel via Incoming Webhook


Install copy this repo to your server and run it. It runs with node and has been tested with PM2. If you need encryption, you are responsible for setting up that yourself.

This uses hapi.js as a simple and fast api solution.

Most of the configuration is done with the config.json file.

```
{
  "server" : {
    "host" : "localhost",
    "port" : 5000,
    "required_headers" : [
      { "key" : "user-agent",
        "values" : ["Slackbot 1.0 (+https://api.slack.com/robots)"] },
      { "key" : "host",
        "values" : ["api.example.com"] }
    ]
  }
}

```

- *server.host* : the hostname passed to the hapi.js server
- *server.host* : the port passed to the hapi.js server
- *server.required_headers* : an array of required headers checked when a request is received. the header must be at least one of the values. this allows for multiple services to submit requests.


```
{
  "auth" : {
    "bot" : [ "USLACKBOT" ],
    "user" : {
      "admin" : [ "UXXXXXXXX", "UYYYYYYYY" ],
      "social" : [ "UAAAAAAAAA", "UXXXXXXXXXX" ]
    }
    "team" {
      "testers" : [ "TXXXXXXXXX" ]
    }
  }
}
```
- *auth.user* : this is used when you specify user *type* security, inside are the groups of users
- *auth.team* : this is used when you specify team *type* security, inside are the groups of teams
- *auth.user.admin* : this is permissions type user, permissions group admin, an array of admin user ids
- *auth.team.testers* : the ids of teams that are used when checking team permissions
- *auth.bot* : the user ids of bots that are used when throwing out bot sent information, this is used to ignore bot information

These names correspond with the security level. If you want to make a new security level, add it to this and to the service.


```
{
  "messages" : {
    "access_denied"   : "Access Denied",
    "request_denied"  : "Request Denied",
    "illegal_command" : "Illegal Command",
    "invalid_request" : "Invalid Request",
    "service_disabled": "Service Disabled",
    "message_failed"  : "Message Failed"
  }
}
```

- *messages* : these are messages that are used as responses in failed cases


This next bit is used for all the services. Currently they are "cli", "devtools" and "xteam". It is used when routing commands and checking permissions so that the logic for checking doesn't have to be implemented in individual services.

```
{
    "servicename" : {
        "enabled" : true,
        "tokens" : "[ThisIsYourSlackServiceToken]",
        "permissions" : "admin",
        "file_path" : "./services/cli.js",
        "post_path" : "/slack/cli/",
    }
}
```

- *servicename.enabled* : when false it rejects requests to this service
- *servicename.tokens* : this is used to validate that outgoing webhooks have the right token (use the token you get when you create the outgoing webhook, you can have multiple tokens for multiple hooks using the same service)
- *servicename.permissions* : admin means only admins in the admin list, team means only teams in the team list, any means any requests get passed
- *servicename.file_path* : this is the location of the implemented service
- *servicename.post_path* : this is the path for the URL to use with the outgoing webhook

## CLI service

```
{
  "cli" : {
    "filters" : {
      "whitelist" : [ "pm2" ],
      "blacklist" : [ "sudo", "&&", ";", "|" ],
      "output" : [
                    {
                      "regex" : "(\\[[0-9]*m)|(┌.*┐)|(├.*┤)|(└.*┘)",
                      "txt" : ""
                    }
                  ]
    },
    "commands" : [
      { "key" : "restart all",
        "value" : "pm2 restart all" },
      { "key" : "node processes",
        "value" : "ps -aux | grep node" }
    ],
  }
}
```

- *cli.filters.whitelist* : commands will only be executed if you whitelist the command. for example, if pm2 is whitelisted you can run any of the normal pm2 commands like "pm2 list", "pm2 restart 0" or "pm2 delete 0"
- *cli.filters.blacklist* : these are checked first looking for banned words. use this to block specific uses and to prevent command chaining
- *cli.filters.output* : this is an array of regex filters that are executed on the output of the commands, currently set only for removing the command line colors and some weird symbols
- *cli.commands* : these are aliases for proper cli commands. notice you can do any command chaining here. the working directory is the same as the application


## Dev Tools service

This service is simple and has little to configure. It is used to get Slack specific ids. The commands you can use with devtools are:

- userid : returns your user_id
- channelid : returns the current channel_id
- teamid : returns the current team_id
- serviceid : returns the current service_id

You'll find these values in the posts sent by Slack's outgoing webhooks.

## X Team service

This is a message relay service that can even relay messages across teams or just relay messages to other channels. It relies on an incoming and outgoing webhook.

To set this up, it relies solely on you setting up your webhooks. First, create an incoming webhook with slack in the channel you want to relay messages into. It will give you something that looks like this as a URL

```
https://hooks.slack.com/services/T0XXXXXXX/B0XXXXXXX/someuglytokenlikething
```

You are going to take the last part and use it with the outgoing webhook. For the channel you want to use to relay content to your incoming webhook, this could be in your team or another team, the outgoing webhook will be configured with this URL:

```
http://api.example.com/your/xteam/path/T0XXXXXXX/B0XXXXXXX/someuglytokenlikething
```

Then anything that meets your outgoing webhook requirements will be relayed to the channel specified by the incoming webhook.

You shouldn't touch the config.

```
{
  "xteam" : {
    "host" : "hooks.slack.com",
    "path" : "/services"
  }
}
```

- *xteam.host* : this is the host for the incoming web hook (you don't need to change)
- *xteam.path* : this is the path for the incoming web hook (you don't need to change)


## Twitter

This is built to let you tweet from Slack. However, it is built using things from the twitter developer API. You will need to get a dev account there in order to have the proper keys and tokens. And you'll also want a bitly account to use with URL shortening.


```
{
    "bitly_user" : "bitlyuser",
    "bitly_token" : "bitlyreallylongtoken",
    "twitter_consumer_key" : "consumer_key",
    "twitter_consumer_secret" : "consumer_secret",
    "twitter_access_token" : "access_token",
    "twitter_access_token_secret" : "access_token_secret"
}
```

Get that information, complete the config and then you can say (I setup my Outgoing webhook to use "tweet" as the trigger word):

```
tweet I love Slack!
```

Anything after "tweet" will be tweeted and it will respond to tell you exactly what was tweeted. This is not related to the offical twitter integration on Slack. I'm fairly sure in that one you can't tweet and beyond that, this is adding a permissions level to it so you can designate who can tweet from your company account.

TODO: Make bitly optional

## Mandrill Notify

This is built to simply relay your Mandrill Webhooks into a Slack channel. To do this, you'll need to set up an incoming webhook in Slack. It will give you an ugly looking URL like:

```
https://hooks.slack.com/services/T0XXXXXXX/B0XXXXXXX/someuglytokenlikething
```

You'll then use the last 3 parts to configure your path inside the config:

```
{
   "host" : "hooks.slack.com",
   "path" : "/services/T0XXXXXXX/B0XXXXXXX/someuglytokenlikething"
}
```

Then you need to setup your webhook in Mandrill to send to the "post_path" in the config.



## TODO:

Add tests. Hapi JS has a really nice way to do testing, they are pushing requests to the server locally. Will implement tests using mocha.
