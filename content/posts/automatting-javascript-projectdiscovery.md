+++
title = 'Automating JavaScript Secret Scanning with ProjectDiscovery toolkit - Simple Techniques #1'
date = 2024-03-28T16:25:49-03:00
draft = false
description = "A simple and practical guide on how we can automate JavaScript file scanning with ProjectDiscovery tools"
tags = ['pentest', 'javascript', 'automation', 'bug bounty']
+++

## Introduction
Hello guys, I hope you're doing well. I am Daniel (a.k.a oppsec), a pentester from a Brazilian company and a bug hunter in my free time.
I would like to share with you some interesting automation that I've done with the [ProjectDiscovery](https://github.com/projectdiscovery/) toolkit.

For you guys that don't know, **ProjectDiscovery** is the team responsible for the development of various tools like Nuclei, Httpx, Katana, DnsX, and a lot more... I really like the entire 
project and the intention of making some pentest processes easier and automated is so useful. I've been using Nuclei, Httpx, and other toiols by him and everything always worked fine for me.

Basically, I will show how can you combine some of ProjectDiscovery's tools and automate the JavaScript crawling and secret scanning by combining Subfinder + Httpx + Katana and Nuclei (with nuclei-templates).

{{< figure src="https://compote.slate.com/images/d821ec33-6e11-4cf1-a5e1-64e48f0173b7.jpeg" >}}

___

## Setup

To start, you need to install all the tools that we're going to use. Expecting you already have Go 1.21+ installed on your machine, there are the commands to install the tools:

- Nuclei
```
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
```

- Subfinder
```
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
```

- Httpx
```
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
```

- Katana
```
go install -v github.com/projectdiscovery/katana/cmd/katana@latest
```

Make sure to have the `GOPATH` environment variable defined in the correct place so you can call the tools from your terminal.

> To install nuclei-templates you need to execute nuclei cmd at least one time.

___

## The Begin

After installing all the tools, we can start using them and make some magic 🧙. To begin, we need to enumerate our target subdomains to after that capture some endpoints, we're going to use Subfinder to enumerate all the subdomains. I recommend you guys [set up some API keys that Subfinder can work with](https://highon.coffee/blog/subfinder-cheat-sheet/), that's gonna help you find some more subdomains than the default one. 

The command we're going to use is **`subfinder -all -d target.domain -o domains.txt`**. We can expect some result like this:
![subfinder](https://i.imgur.com/JaUc6vx.png)

After enumerating all the subdomains and saving the output to a text file, we need to check which one is alive or not. You can use this command: **`httpx -l domains.txt -o domains.httpx`**

*Tip: You can specify custom ports like 8080, 8081, 81, and others to find other services that are running on the same host.*

{{< figure src="https://i.imgur.com/OGKJUFH.png" >}}

This is one of the principal processes that we're going to use to get the JavaScript files. We will use Katana to capture the endpoints from the subdomains that we found before and search for JavaScript files with the `-jc` flag. I will use this command, but you can improve him with grep and other binaries: **`cat domains.httpx | katana -jc -o endpoints.txt`**

{{< figure src="https://i.imgur.com/pOE0fc3.png" >}}

With all endpoints captured, we are going to filter by the endpoints that end with *.js*, we can use grep: **`grep '.js' endpoints.txt > js_files.txt`**. The problem here is that will take *.json* files too, so we can remove the endpoints that end with *.json* with grep **-v** flag: **`grep -v '.json' js_files.txt > js_files_cleaned.txt`**.

{{< figure src="https://i.imgur.com/uaQUI3v.png" >}}
{{< figure src="https://i.imgur.com/acyk9Ou.png" >}}

After that, we need to save all the JavaScript files. We’re going to use *cURL* and *xargs* binaries to save the files in our directory. We can also use awk to get the domain name and the file name to identify the file after the scan. Before executing the command below, make sure to create a new directory to save all the js files, as you can see, I'm using **`../`** assuming we're already in another directory
```
while read url; do
  filename=$(echo $url | awk -F/ '{gsub(/[\/:]/, "-", $3); print $3 "-" $NF}')
  curl "$url" > "${filename}.js"
done < ../js_files_cleaned.txt
```

{{< figure src="https://i.imgur.com/CErGIC0.png" >}}

___

## The final

The final result of the command is gonna be something like this:

{{< figure src="https://i.imgur.com/VIvgXOx.png" >}}

Now, to find the possible leaked secrets on the code, we're going to use nuclei + nuclei-templates. That's the command I've been using and everything worked fine for me.
```
find <js_files_path> -type f -name "*.js" | while read jsfile; do nuclei -t ~/nuclei-templates/file/ -target "$jsfile" -severity low,medium,high,critical -et ~/nuclei-templates/file/xss/dom-xss.yaml; done
```

{{< figure src="https://i.imgur.com/TjDh4oE.png" >}}
{{< figure src="https://i.imgur.com/a5hZ5ql.png" >}}

___

## Warnings and Final Thoughts
- You’re probably going to be blocked by the WAF, so be careful.
- Of course, they're gonna be so many false positives so be patient and check if the tool report is valid.
- There are probably so many better ways to do this, but I think this method is a good way too. Use it if you want.
- I'm not sharing this to prove anything, I'm sharing because I think is gonna be useful to people who want to try other things.

___

## References
- [Subfinder Cheatsheet](https://highon.coffee/blog/subfinder-cheat-sheet/)
- [ProjectDiscovery](https://github.com/projectdiscovery)
- [Davis Sopas -  Using katana to get API endpoints](https://www.youtube.com/watch?v=9d1D-DB6BkE)

### Tags