---
layout: post
title: "JSMon: Automated JavaScript file monitoring"
categories: hacking pentesting bugbounty recon web js
---

## JSMon: Automated JavaScript File Monitoring
Today I'm proud to release [JSMon](https://github.com/robre/jsmon), an online change monitoring tool for javascript files!

To find new bugs quickly, it is a **key capability** to be able to **identify new features** and changed code as soon as they are pushed to the public. In modern web applications, features are typically implemented with javascript - making JS files a prime target for monitoring!

The idea for this tool is to continuously fetch a given list of javascript files, while keeping a database of all seen versions of those files. When a new version appears, **JSMon** saves the new version of the file, and notifies the user via IM using the Telegram API. The notification includes a beautified diff, so that one can easily spot changes even in minified code.
Continuous scanning is realized by simply running the script via cronjobs. 

When a monitored file changes, JSMon will send you a notification that looks like this:

![](/assets/jsmontelegram.png)

As you can see, it tells you not only what file changed, but also the filesize, and a ``diff.html`` file! This diff file contains a nice view of all changes to the javascript file that you are monitoring!
The diff looks like this:

![](/assets/jsmondiff.png)

One cool thing: Even for minified scripts a good looking diff is generated, by beautifying the script files!

**JSmon** includes a `downloads/` directory for the downloaded files, a `targets/` directory for lists of urls to fetch, and a `jsmon.json` file that serves as the database connecting the target urls to downloaded file

### Example

To actually use JSMon, you need to follow a few simple steps:

#### Installation

simply run 
```bash
git clone https://github.com/robre/jsmon.git 
cd jsmon
python setup.py install
```
to clone the git directory and install JSMon.

#### Setup

For JSMon to run as intended you need to first setup your Telegram API keys, and a cronjob.

To get your Telegram API key and chat_id, you can follow [this blogpost](https://blog.r0b.re/automation/bash/2020/06/30/setup-telegram-notifications-for-your-shell.html).
You need to save the API key and chat_id inside `jsmon.py` - simply look for the `CHANGEME` placeholders at the beginning of the script.

Now JSMon should be useable, all you need to do is create a cronjob to run it regularly! To create a cronjob you run 
```bash
crontab -e
```
You will be prompted with an editor where you need to create an entry for JSMon in the cronjob syntax. To create a job that runs once a day, simply put 
```bash
@daily python /PATH/TO/jsmon.py
```
(don't forget to change the path to wherever you installed jsmon!)

Altenatively, if you want JSMon to run once an hour, you can use `@hourly` instead.

Save the file and you're ready!

#### Adding Targets
The last thing you need to do, is to add some targets for JSMon to fetch. JSMon will look for files in the `targets/` directory and try to fetch each line from each file. So, to monitor my testing js file: `http://r0b.re/test.js`, you can simply run `echo "http://r0b.re/test.js" >> targets/test-r0bre` to create a file named `test-r0bre` in the `targets/` directory, and insert the script url in it.
Now you can run `python jsmon.py` and JSMon will add the testfile to the database! The next time this file changes you will get a notification on Telegram containing a diff of the files so you can quickly see what changed!

### Conclusion

That's all you need to do to setup your own JS monitoring solution. If you have questions, feature requests, or any feedback feel free to contact me!


[@r0bre](https://twitter.com/r0bre), 5. July 2020

