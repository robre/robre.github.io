---
layout: post
title: "Hunting for Endpoints"
categories: hacking pentesting bugbounty recon web
---

## Hunting for Endpoints

When hunting for new bugs on the web, be it for bugbounties, pentests, or other engagements, it is crucial to have excellent recon. Especially when thousands of other bughunters are targeting the same public bugbounty program, or if you simply want to be as thorough as possible for a black box pentest. This is the case for subdomain enumeration (for which you can find an excellent methodology at [0xpatriks blog](https://0xpatrik.com/subdomain-enumeration-2019/)), but also for endpoint enumeration on a given subdomain. More endpoints correspond to a higher attack surface, meaning more bugs! 

![endpoints-bugs](/assets/endpoints-bug.png)

In this blogpost I want to help you optimize your recon process to find as many endpoints as possible, that others don't.

### Basics

There are multiple ways to enumerate endpoints of a webapp:

- manual browsing
- spiders/crawlers
- directory bruteforcing (aka "dirbusting")
- databases such as google or waybackarchive
- and code auditing

One classic trick includes checking out the ``http://SOMEWEBSITE.com/robots.txt`` file. This file is used to tell search machines (aka robots), such as google, which endpoints should be indexed, and more importantly, which ones should be ignored. This can often include interesting endpoints that the target doesn't want to pop up in a google search.

#### Manual Browsing
This method is the simplest - simply attach your webbrowser a logging proxy such as Burp, and start browsing the target website. While browsing, try to perform as many actions on the website as possible - simply click on anything you can. Meanwhile, burp will build a sitemap for you in the background. You can export the links in Burp by going to Target Tab -> right click a subdomain -> Copy URLs in this host / Copy links in this host. This method is effective and simple, however you might accidentaly miss some endpoints - thats where a spider/crawler comes in.

#### Spiders/Crawlers
Spiders/Crawlers automate the aforementioned process. Starting with a root page, they will automatically visit each link on the page, and continue this process recursively until either no new links are found, or a certain recursion depth is reached. Burp has an integrated spider for this task.

#### Dirbusting
Dirbusting is the process of simply taking a list of possible endpoints, and trying each one on the target to see if it exists. For instance, endpoints such as `/api/`, or `/admin/` are quite commonplace - therefore it might be a good idea to just test them. Dirbusting automates this process with huge lists of common endpoints, and postprocessing filters to assess if a tested endpoint is valid, or not. Dirbusting will be the main focus of this blogpost, because it is the best way to find endpoints that others don't, due to the fact that it largely depends on the used settings when performing it.

#### Public Databases
Another effective method to find endpoints is the use of search engines and public databases, such as the waybackarchive. Search engines can be queried manually using dorks such as `site:`, `inurl:`, and many more. This way one may find sites that were indexed by google, code that accesses specific endpoints (search for `site:github.com SOMEWEBSITE.com`), or forumposts with a link referencing a legacy api - anything is possibe! The [waybackarchive](https://web.archive.org/) is a database that archives the history of the internet, often indexing legacy endpoints that might be overlooked otherwise. It's always worth a try to check it out. There also exists a [handy tool](https://github.com/tomnomnom/waybackurls) by [@TomNomNom](https://twitter.com/tomnomnom) to automate this lookup process!

#### Code Auditing
This method includes checking out either open source projects on github that reference the target website (can be found using google, or github search), or reverse engineering and auditing the code of any standalone applications provided by the target. Such apps can be desktop applications, mobile apps, or even hardware devices. While this can sometimes be done quite simply by just running `strings targetapp`, or by using a proxy to log all web requests, this method oftentimes involves significant reverse engineering efforts which is out of scope for this blogpost. The simplest approach to code auditing for webapps however is the auditing of the sites included javascript files. This is a very effective technique especially when paired with dirbusting for javascript files ;)
Just to give an example: Performing dirbusting on the `/js/` directory for `staff.bountypay.h1ctf.com` at the recent hackerone CTF competition, yields the file `/js/website.js`, with the following contents:

```javascript
$('.upgradeToAdmin').click(function () {
  let t = $('input[name="username"]').val();
  $.get('/admin/upgrade?username=' + t, function () {
    alert('User Upgraded to Admin');
  });
}), $('.tab').click(function () {
  return $('.tab').removeClass('active'), $(this).addClass('active'), $('div.content').addClass('hidden'), $('div.content-' + $(this).attr('data-target')).removeClass('hidden'), !1;
}), $('.sendReport').click(function () {
  $.get('/admin/report?url=' + url, function () {
    alert('Report sent to admin team');
  }), $('#myModal').modal('hide');
}), document.location.hash.length > 0 && ('#tab1' === document.location.hash && $('.tab1').trigger('click'), '#tab2' === document.location.hash && $('.tab2').trigger('click'), '#tab3' === document.location.hash && $('.tab3').trigger('click'), '#tab4' === document.location.hash && $('.tab4').trigger('click'));
```
See the obvious highly interesting endpoints? :D
Also you can use [Linkfinder](https://github.com/GerbenJavado/LinkFinder) to automatically grab endpoints from javascript files!

### Tooling

A major factor for finding endpoints is the correct tooling. I'm convinced that Burp is probably the best tool to accompany the manual methods, and it also incorporates a spider which I personally use. I have not much experience with other spiders, therefore I wouldn't feel comfortable giving any recommendations for any free options here - maybe you can send me some suggestions on [twitter](https://twitter.com/r0bre) and I might add them here :)

For dirbusting there exist a number of tools, among them the classic but outdated *dirbuster*, its console variant *dirb*, and multiple alternatives, such as *dirsearch* and the much improved versions *wfuzz*, *gobuster*, and my (current) favorite, *ffuf*. I highly recommend using either *gobuster* or *ffuf*, as they typically outperform the alternatives, due to being written in go, with a focus on performance and allowing high thread counts. 
Another hot tip: dark-warlord14 created this [handy script](https://github.com/dark-warlord14/ffufplus) to automate and optimize some common tasks with *ffuf*!

### Scan settings

Now all this information goes to waste if you go on and just run *ffuf* with the standard settings - it will only get you the default results that everyone else gets. So let's take a closer look to determine how to scan correctly.
Let's start with a default ffuf scan: 
``ffuf -w wordlist.txt -u https://r0b.re/FUZZ``
Per default, ffuf accepts the response codes 200,204,301,302,307,401,403. This misses important status codes such as 500 Server Error, which might happen when we send a request with missing parameters, or 405 Method Not Allowed, which indicates that we should maybe try a POST (or other) request. I like to add these status codes (using `-mc 200,204,301,302,307,401,403,500,405`), or simply include all status codes (using `-mc all`) and do the filtering later.

The next option is turned on by default in *ffuf*, but turned off in *gobuster*: Extra response information, especially the response length. When we get a `200 success` status code from dozens of sites with the exact same response length, it might be a default error page with the wront status code - including the response length in the output makes it easier to spot such anomalies. To turn this option on in gobuster, use `-l`

Next, oftentimes a site might return a `403` Status Code when trying to access a directory, such as `/js/`, but 200 when accessing a file within the directory, e.g. `/js/jquery.min.js`. Therefore it is important not to stop when finding an endpoint, but to recursively keep searching on the endpoint. For this, ffuf has the `-recursion` option, which should be used with a defined recursion depth (`-recursion-depth 2`), so that the scan won't take forever. Another option to limit the duration is the `-maxtime-job 120` switch, in this case setting the maximum time for a job, such as following an endpoint recursively ,to 120 seconds.

The last option that is quite important is the extensions that are used. There are a lot of interesting extensions to try, such as `.php,.asp,.html,.js,.min.js,.db,.bak,.txt,.py,...` and much more. Typically I will only use a small subset, depending on what technology the website uses - if it's a website running php on a linux system it doesn't make much sense to look for `.asp` files. This option needs to be adjusted to your needs and to what you are looking for. Keep in mind that each extension increases the multiplier for the number of total requests by one!
With all the options considered, we might end up at the following command:
```
ffuf -w customwordlist.txt -u https://r0b.re/FUZZ -mc 200,204,301,302,307,401,403,500,405 -E .php,.txt,.db,.php.bak,.html,.md -recursion -recursion-depth 2 -maxtime-job 300
```
### Wordlists

If you really want to find endpoints that others don't - the most effective tool is a custom wordlist. If you have a wordlist with better, or just more entries, you will find more endpoints then others. It is however important to find the right balance in the length of the wordlist - a short wordlist won't find all endpoints, while a overly long wordlist might just waste your precious time with unlikely endpoints.
Oftentimes I see people just using ``SecLists/Discovery/Web-Content/common.txt``, or `big.txt`. But this is not sufficient. There are other good wordlists in `SecLists`, such as raft, or robots disallowed, but my tip is not to use a single wordlist - combine them! Also, when combining two wordlists, keep in mind to make them uniform, so that you can remove duplicates with `sort | uniq` easily - I like to have all endpoints in lowercase, with no trailing or leading slashes. This can be accomplished with the following oneliner:
```bash
cat wordlist1 wordlist2 wordlist3 | tr A-Z a-z | sed "s,^/,," | sed "s,/$,," | sort | uniq > mywordlist.txt
```
oftentimes numeric entries take up huge space in wordlists and don't yield good results - therefore I like to sometimes remove them using `grep wordlist | grep -vE '\d\d\d\d\d\d?'` for 5- and 6-digit numbers
Another tip when constructing wordlists - create and use target specific wordlists. If you know you target uses SAP, maybe try the `SecLists/Discovery/Web-Content/sap.txt` wordlist, or create one from public sources. Maybe you have manually encountered a list of endpoints on another subdomain for the same target org - you can export them from burp into a custom wordlist and use that! Be creative, this is where you get results that others don't.

### Optimizations
Some optimizations to consider when doing large scans:

- Split your huge wordlist into two: One containing the most common endpoints, and the other containing the rest. This way you can first run the small one to look for low hanging fruit quickly.
- Create separate wordlists for separate scan settings - maybe one wordlist with directory names that you scan without extensions, and another wordlist with file names that you scan with every imaginable file extension
- Create aliases for your common settings - this is a no-brainer!
- Don't forget to set the thread count to something your computer can handle

![acceleration](/assets/acceleration.png)

### Conclusion

I have outlined some ways to get the most out tools such as *ffuf* and explained my approach to endpoint hunting. My key points:

- Use a combination of methods to find as many endpoints as possible
- Use the correct settings for the tools you use, know what they are doing!
- Use custom wordlists for optimal results
- Optimize your workflow!


[@r0bre](https://twitter.com/r0bre), 10. June 2020