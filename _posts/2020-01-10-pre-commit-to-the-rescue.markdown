---
layout: post
title: pre-commit to the rescue
tags: [pre-commit, python, black, git hooks]
categories: python, pre-commit
date:   2020-01-10 13:37:00 +0200
excerpt_separator: <!--more-->
---

During a keynote on last years [PyCON Berlin](https://pycon.de) somebody mentioned that he was watching that guy on [twitch](https://www.twitch.tv/) code for an audience. While hesitent at first, I actually followed [@anthonywritescode](https://twitch.tv/anthonywritescode) / [@asottile](https://github.com/asottile) and checked him out the next weekend. While the amazement of watching other people, especially proficient people, code live and answer question in real time is a topic in itself,what I found especially interesting was what he was working on - something called [pre-commit](https://pre-commit.com/).

<!--more-->

Raise your hand if you have accidentally committed code that does not compile, or an invalid json without that nasty closing bracket.

I'm sure you haven't, but I most certainly did. 

### git hooks
The whole concept of git hooks is new to me. What they do is to perform actions before or after `git commit` or before a `git push`. So if I commit, a hook will check my commit for valid json, yml or compilable code. Well, that's brilliant. No, more unnecessary failing ci/cd pipelines. Hooray! The way I understand it tho, is that before pre-commit, git hooks were project specific and resided in your .git folder, or they're global, tedious to setup and share among contributors of your projects.

### enter pre-commit
Pre-commit aims to solve this by managing the installation and execution of hooks. You can either use one of the [out-of-the-box hooks](https://github.com/pre-commit/pre-commit-hooks) or custom hooks ones, e.g. for [black](https://iwpnd.pw/articles/2020-01/black-python-code-formatter).

### installation
The installation is as easy as

```
pip install pre-commit
```

Then you create a ```.pre-commit-config.yaml```. You can do this either on your own, or use the sample config that comes with it executing 
```
pre-commit sample-config
``` 

What I happen to use is this:

```yaml
repos:
    -   repo: https://github.com/pre-commit/pre-commit-hooks
        rev: v2.3.0
        hooks:
        -   id: check-yaml
            args: [--unsafe]
        -   id: end-of-file-fixer
        -   id: trailing-whitespace
        -   id: check-json
        -   id: check-added-large-files
        -   id: check-ast
    -   repo: https://github.com/psf/black
        rev: 19.3b0
        hooks:
        -   id: black
```

All the config does, is specifying the repositories that contain the hooks you want to use, and optional args you can add to specific hooks. After you setup your config, you attach the hooks to your repository using:

```
pre-commit install
```

From now, when you want to `git add` and `git commit` a diff, pre-commit will perform the checks you specified in your `.pre-commit-config.yaml`.

![pre-commit workflow](/img/2020-01-10-pre-commit.png)

If all checks pass, your commit goes through. If checks fail, or hooks modified your files during the checks, you either have to edit, add and commit them again, or automate that part. `check-json` for example has an `--autofix` argument, that automatically attempts to fix your brokes .json files before they're committed.

That's it!