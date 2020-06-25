---
layout: post
title: "Crafting a Custom Wordlist for Flask Webservers"
categories: hacking pentesting bugbounty recon web python bash
---

## Crafting a custom wordlist for python-flask webservers

A while ago I came to the conclusion, that custom wordlists for content discovery are one of the best ways to get ahead in Bug Bounty. They allow you to find more servers and endpoints, widening the attack surface. Then, a couple weeks ago, [TomNomNom](https://twitter.com/tomnomnom) held a great talk about custom wordlists at NahamCon, giving some amazing tips and insights on his process of creating a custom wordlist (You can find the talk [here](https://www.youtube.com/watch?v=W4_QCSIujQ4)). Today I saw a tweet by [@d0nutptr](https://twitter.com/d0nutptr) sharing a [custom wordlist](https://forum.bugcrowd.com/t/dropping-a-cool-wordlist/9211) that was scraped from popular go repositories, so I decided to do the same for python!

### Routing in python-flask

I decided to go for [python-flask](https://flask.palletsprojects.com/en/1.1.x/), because that's the library that I am most familiar with when it comes to writing a webserver. Our plan here is to extract routes from flask files, so that we can create a wordlist containing endpoints that are known to be used by flask developers. To extract the endpoint routes, we need to know how they are implemented. In flask, this is done using the `@app.route('/someroute')` decorator followed by the function implementing the logic for that endpoint. So what we actually only need to extract this decorator line from the projects!

### Scraping github

Now that we know what we are looking for, we can search github for python projects containing `@app.route`. In the github search query language, our search would look like this: `@app.route+in:file+language:python+extension:py+flask`. `in:file` tells github to look inside of files, `language:python` and `extension:py` specify filetypes, and for good measure, I added the `flask` keyword to the search.
To perform the github search automatically, we can use the github API. Keep in mind that github limits searchresults to 1000 per search. Using the `&page=` and `&per_page=100` (which is the max page size) params, we can get 100 results per API call, so that we need a total of 10 requests. I wrote a little function that does this: (remember to use your credentials)
```bash
search_github(){
  searchterm=$1
  for (( page=1 ;page < 11; page++)); do # get 10 pages with 100 results each = 1000 results
    sleep 10 # Be nice to github
    curl -u "username:password" https://api.github.com/search/code\?q\=${searchterm}\&page=${page}\&per_page=100 | tee -a gitsearch.txt
  done
}
```

Github returns a json response, which contains the searchresults as a list under the `items` key. Each item contains some information, including a direct link to it under the `html_url` key. If we want to download the file, we need to convert this url to a raw githubusercontent url, which we can easily do with sed. To retrieve all file urls I created this oneliner:
```bash
cat gitsearch.txt | jq -r '.items[].html_url' | sort -u | sed  's/github\.com/raw.githubusercontent.com/' |sed 's,/blob,,' > rawfiles.txt
```
Next, we need to download these files and extract the line that we are interested in - `app.route` remember? We can do this easily like this:
```bash
cat rawfiles.txt | while read line; do
  echo $line
  curl -s $line 2>&1 | grep 'app\.route' | tee -a routes.txt
done
```
### Extracting routes

We're almost there! Now we have a list of calls to `app.route`, where we want to extract the routes from. For this I used yet another oneliner:
```bash
cat routes.txt | tr -d "[:blank:]" | grep -E '^@app\.route\(.*\)' | tr '"' "'" | cut -d "#" -f 1 | sed -s 's/,.*$/)/' | cut -d "'" -f 2 |sort -u > flaskwordlist.txt
```
Here we use another grep to remove any false positives. `tr` is used to remove all whitespace, and then to translate all double quotes to single quotes, so that all results are coherent. Next, we remove all comments using `cut`, and using `sed` we can take decorators that also specify a method, such as `@app.route('/', methods=['GET', 'POST'])`, and normalize them to simply `@app.route('/')`. In the end we select the content between the first pair of single quotes, which is exactly the routes that we want to extract. Done!
We end up with a nice wordlist containing common routes used in flask projects. Note that some routes contain `<someVariable>` directives, indicating where a User can supply variables. We can replace them with some arbitrary values when using the wordlist.

### Summary

I created a little script to summarize all the oneliners:

```bash
#!/bin/bash

search_github(){
  searchterm=$1
  for (( page=1 ;page < 11; page++)); do # get 10 pages with 100 results each = 1000 results
    sleep 10 # Be nice to github
    curl -u "username:password" https://api.github.com/search/code\?q\=${searchterm}\&page=${page}\&per_page=100 | tee -a gitsearch.txt
  done
}

search_github "@app.route+in:file+language:python+extension:py+flask"
cat gitsearch.txt | jq -r '.items[].html_url' | sort -u | sed  's/github\.com/raw.githubusercontent.com/' |sed 's,/blob,,' > rawfiles.txt
count=$(wc -l rawfiles.txt|cut -d " " -f1)
echo "getting $count files"
cat rawfiles.txt | while read line; do
  echo $line
  curl -s $line 2>&1 | grep 'app\.route' | tee -a routes.txt
done
cat routes.txt | tr -d "[:blank:]" | grep -E '^@app\.route\(.*\)' | tr '"' "'" | cut -d "#" -f 1 | sed -s 's/,.*$/)/' | cut -d "'" -f 2 |sort -u > flaskwordlist.txt
```

You can download the resulting wordlist [here](/assets/flask-wordlist.txt)

[@r0bre](https://twitter.com/r0bre), 25. June 2020
