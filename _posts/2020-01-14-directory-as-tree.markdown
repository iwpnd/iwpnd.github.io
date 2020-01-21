---
layout: post
title: Save a directory tree asci representation to be used in documentations
tags: [homebrew, documentation]
categories: homebrew macos documentation
date:   2020-01-14 13:37:00 +0200
---
I was looking for an easy way to build a tree-structure from an input dictionary that I could use in documentation e.g. on github and stumbled upon [tree](http://mama.indstate.edu/users/ice/tree/), a homebrew formulae.

Setup on macOS is as easy as:

```
brew install tree
```

Now if you `cd` into a directory that you want to map into a visually representation in ascii, then you would just type

```
tree . >> output.txt
```

which would return:

```bash
├── Dockerfile
├── LICENSE
├── README.md
├── app
│   ├── __init__.py
│   ├── api
│   │   ├── __init__.py
│   │   └── api_v1
│   │       ├── __init__.py
│   │       ├── api.py
│   │       └── endpoints
│   │           ├── __init__.py
│   │           └── endpoint.py
│   ├── core
│   │   ├── __init__.py
│   │   ├── config.py
│   │   └── models
│   └── main.py
├── pyproject.toml
├── requirements.txt
├── scripts
├── template.yml
├── tests
│   └── __init__.py

8 directories, 17 files
```

Something similar could be achieved in Python with this [answer on Stackoverflow](https://stackoverflow.com/a/49912639).