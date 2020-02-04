---
layout: post
title: rapidly build and deploy applications with streamlit and docker
tags: [python, streamlit, docker]
categories: python streamlit docker
date: 2020-02-04 10:37:00 +0200
toc: true
---

Ever since May 2019, funny enough right AFTER my wedding, I decided to lose weight. Spoiler, the endeavor has been successful, but this is hardly blog-worthy. So, I was looking to apply what I've learned during my last sessions with [FastAPI](https://iwpnd.pw/articles/2020-01/deploy-fastapi-to-aws-lambda) on a new project, preferably one that made use of a database this time, and that I could use daily if I wanted to. The result is [weightwhat](https://github.com/iwpnd/weightwhat), a CRUD API using both FastAPI and PostgreSQL to document my weight-loss journey. A day's work and I had this thing up and running, so far so good, and even learned a couple of new things about docker-compose. Now, while I already talked about a big plus of FastAPI, the automatic [Swagger UI that is created with your application](https://iwpnd.pw/articles/2020-01/opinion-on-fastapi), I felt that some kind of frontend with some visualizations to show the progress wouldn't hurt. While in the past I did something like that with [Flask](https://github.com/pallets/flask) and [Jinja](https://github.com/pallets/jinja), I feel that visually and functionally those efforts were [underwhelming](https://github.com/iwpnd/geojsonconverter) to say the least. Adding some frontend development skills to my skillset is eventually on the list, but for now, I wanted to have a solution that is in a more immediate reach.

## Enter streamlit

> "Streamlit is an open-source app framework for Machine Learning and Data Science teams. Create beautiful data apps in hours, not weeks. All in pure Python. All for free." [streamlit](https://www.streamlit.io/)

Okay, that sounds promising.

### I like me some sprinkle

From the documentation, it says to "sprinkle a few Streamlit commands" into a Python script and run the script.

```python
import streamlit as st

name = "Ben"
st.title(f"Hello {name}")
st.subheader("How are you?")
```

```bash
$ streamlit run app/main.py

> You can now view your Streamlit app in your browser.

 Local URL: http://localhost:8502
 Network URL: http://192.168.178.21:8502
```

<p align="center">
<img src="/img/2020-02-streamlit/streamlit1.png" alt="streamlit in browser">
</p>

That's it? That's it! Once the script is executed streamlit will spawn a server in the background that takes your script and renders the output. You can reach it via `localhost` in your home network or your companies network to quickly share it with your colleagues or to check how the application looks on your mobile device without going through the effort of deploying it.

Let's add a little more to it.

```python
import streamlit as st

name = st.text_input("What's your name?")

if name:
 st.title(f"Hello {name}")
 st.subheader("How are you?")
```

<p align="center">
<img src="/img/2020-02-streamlit/screenrecording1.gif" alt="streamlit in browser">
</p>

So what can we unpack here? We have some interactivity, and we have conditions for the elements that are displayed. All that is possible with some [magic](https://docs.streamlit.io/api.html#magic-commands). Whenever there is a variable on a single line, or streamlit notices one of its commands, they will be published in your app. So, defining a variable does not print the variable, however putting it on its own, will prompt streamlit to show its value.

### Now what?

Adrien Treuille summed streamlit up in a great [blog post](https://towardsdatascience.com/coding-ml-tools-like-you-code-ml-models-ddba3357eace) on Medium where he goes into more detail on the architecture and some more examples.

I think one can immediately see why streamlit excites me. Never could you faster create an application than now [streamlit](https://www.streamlit.io/) enables you to do. You do not have to rely on your frontend engineers, you don't have to wait for them to allocate some time to your little project. Nor do you have to wait until somebody else can use your model for inference, see your visualization without handing out files or explore your datasets. Sure, you could have that interactivity and visualization in a jupyter notebook, but the fun stops when you have to version control that beast. I'm hooked, are you?

## Developing streamlit inside a container

Until I have tested to deploy a streamlit application in AWS Lambda, I assume that the easiest way is to develop it in a container and ship that, to avoid shipping my machine. For that purpose, I created a [simple template](https://github.com/iwpnd/streamlit-docker-example) for you to try it out.

```bash
# Dockerfile
FROM python:3.7
WORKDIR /usr/src/app
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

COPY ./requirements.txt /usr/src/app/requirements.txt # atleast: streamlit

RUN pip install --upgrade pip setuptools wheel \
 && pip install -r requirements.txt \
 && rm -rf /root/.cache/pip

COPY ./ /usr/src/app
```

```bash
# docker-compose.yml
version: '3.7'

services:
 app:
 build: ./
 command: streamlit run app/main.py --server.port 8501
 volumes:
 - ./:/usr/src/app
 ports:
 - 8501:8501
 image: yourstreamlitapp:latest
```

This tells docker to create a service, from the `dockerfile`, mount your folder to the container, expose the ports and tag that image.
Now bring that service up with

```bash
docker-compose -d up --build
```

This will run the container in `-d` detached mode in the background and build the image if it isn't present on your local machine. You can now reach your application at [http://localhost:8502](). If that doesn't work check if your service is up and running `docker-compose ps`, and if it is running but the application is not displayed you can check the logs like so `docker-compose logs --follow app`.

Since docker-compose is mounting your current directory, your container will always have your current application `main.py`, therefor streamlit will also re-run, if it detects a change made to your application. This way you can incrementally build, test and try your application.
Once you're done: 

```bash
docker-compose down
```

That's it. :ship:

## Weightwhat?

Now, back to that CRUD api that I needed a frontend for. It turns out streamlit is compatible with a lot of the major libraries and frameworks that we all love. I used that change to finally learn a little about declarative visualization with [Altair](https://altair-viz.github.io/). Altair is not only highly customizable but also very well documented and allows for easy to implement interactivity, which is a nice-to-have on an application evolving around data exploration. You can check it out [here](https://github.com/iwpnd/weightwhat).

<p align="center">
<img src="/img/2020-02-streamlit/weightwhat.png" alt="weightwhat">
</p>