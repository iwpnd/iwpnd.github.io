---
layout: post
title: Spellchecking with Grammarly in VSCode
tags: [python, vscode, markdown]
categories: ide
date: 2020-01-28 13:37:00 +0200
# toc: true
---

My last [article on fastAPI](https://iwpnd.pw/articles/2020-01/deploy-fastapi-to-aws-lambda) was the most-viewed article by now, but at the same time the article I had to update the most due to typos and grammatical errors. Sure you can write you texts in something like Pages, or Word, and copy them over to markdown afterward, or save them as markdown (apparently [this](http://www.writage.com/) is a thing now). 
I for one like to write directly in my editor of choice which is VSCode, even though it does not come with spell-checking batteries included. So you could resort back to Grammarly to do the spell-checking for you, but you would still have to copy and paste the text over to from Grammarly and back into your editor. Naah!

Luckily for people like me, there are people out there that have our back on this. Folks like [@znck](https://znck.dev/) who felt the same, reverse-engineered the Grammarly client and build a [VSCode extension](https://marketplace.visualstudio.com/items?itemName=znck.grammarly) to fulfill my every spell-checking need.

All you as a user have to do is to register an account with [Grammarly](https://app.grammarly.com/), install the extension from the marketplace and add 

```json
    {
        "grammarly.username": "you@typo.com",
        "grammarly.password": "typ0"
    }
```

to your `settings.json` in VSCode or manually in the VSCode settings via `Settings > Extensions > Grammarly`. 

There you go, full Grammarly support in your code editor.

<p align="center">
<img src="/img/2020-01-28-grammarly-vscode-1.png" alt="grammarly in vscode">
</p>

Mistakes are presented like your code linting errors, which is pretty neat if you're debugging through the list anyways. You can then either choose to correct them from there like so:

<p align="center">
<img src="/img/2020-01-28-grammarly-vscode-2.png" alt="grammarly in vscode">
</p>

Another possibility is to hover over the text in your editor directly:

<p align="center">
<img src="/img/2020-01-28-grammarly-vscode-3.png" alt="grammarly in vscode">
</p>


Say you know that you never make mistakes in your code,.. because, you know, who does, right? Then you can simply exclude certain parts of your markdown from spellchecking.

```json
    {
        "grammarly.username": "you@typo.com",
        "grammarly.password": "typ0",
        "grammarly.diagnostics": 
            {
            "[markdown]": {
                "ignore": ["inlineCode", "code"]
            }
        }
```

Now if that piqued your interest, go ahead and try it out yourselves. And if you're interested in how the extension was brought to life, there is an extensive [blog post](https://znck.dev/blog/2019-grammarly-in-code) of the author about that. Let's hope that this text is free from typos, at least.
