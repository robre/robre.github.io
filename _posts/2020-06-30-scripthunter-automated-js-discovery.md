---
layout: post
title: "Scripthunter: Automated JS Discovery"
categories: hacking pentesting bugbounty recon web js
---

## ScriptHunter: Automated JS Discovery

Today I am proud to release a little script of mine that is very helpful to find javascript files of web applications. **Scripthunter** uses multiple methods to locate as many js files for a given domain as possible.
It is available on GitHub [HERE](https://github.com/robre/scripthunter).

The general idea for **scripthunter** is that I wanted to run a **ffuf** scan on every single directory that might contain javascript files. That means, if
a website ``A.com`` contains contains the javascript files `A.com/js/main.js` and `A.com/static/js/test.js`, I want to scan `A.com/js/` and `A.com/static/js/`. Additionally I always want to scan `A.com/`. This process is fairly boring - perfect for automation! **Scripthunter** will first run **gau** to find javascript files from public sources for a given domain. Then, it runs **hakrawler** to find JS files that are directly referenced in the websites code, e.g. in `<script src=` tags. From those results, a list of directories containing javascript files is extracted, each of which is scanned using **ffuf**, with a custom wordlist I created just for this. Lastly, because especially **gau** tends to yield some nonresponsive results, all found javascript files are tested for responsiveness using **httpx**. Because these scans can take some time, I included a notification function, that will send a **telegram** message, once **scripthunter** is done.
The notification looks like this:

![telegram](/assets/telegram.png)

The Output of the tool looks like this:

![output](/assets/scriptfinder.png)

## scripthunter-wordlist

To enable **scripthunter** to find as many javascript files as possible, I created a custom wordlist for common javascript filenames. To find such filenames, I used the following strategies:

1. Manual aggregation
2. common js filenames from github
3. crawling top 1000 websites
```bash
cat top_websites.txt | hakrawler -js -depth 1 -scope yolo -plain | unfurl path | rev | cut -d "/" -f1 | rev  | tee -a wordlist-topsites.txt
```

4. cdnjs.com for common library names
```bash
curl https://api.cdnjs.com/libraries | jq -r '.results[].latest' | rev | cut -d '/' -f1 | rev > wordlist-cdnjs.txt
```
Additionally, I manually filtered and normalized the resulting wordlist.

I will try to keep updating the wordlist in the future.


[@r0bre](https://twitter.com/r0bre), 30. June 2020
