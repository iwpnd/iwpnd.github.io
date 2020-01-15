---
layout: post
title: Opinion on fastAPI
tags: [python, fastapi]
categories: python fastapi
date:   2020-01-13 13:37:00 +0200
---
Whenever I wanted people to interface some module I wrote or let them do inference on a model I resorted to [Flask](https://github.com/pallets/flask). After all it is lightweight, widely used, well documented and there are gazillion tutorials that help you out on getting started, e.g. the famous [Flask Mega Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world). 

## the right tool for the job
I didn't really know that I needed something else until somebody told me about [fastAPI](https://fastapi.tiangolo.com/). See, you can build APIs with [Flask](https://github.com/pallets/flask), but it was not specifically intended with that task in mind, but with the ability to construct complex web applications. While that is not specifically excluding, there's still that "right tool for the job" mentality that I want to keep in mind when I build things. 

## things I like about fastAPI
I'm not one to tell you about the nitty-gritty details of both frameworks, and this is certainly not a Flask vs fastAPI discussion. I can however tell you what I like about fastAPI, you take a look at it and decide for yourself.

- **documentation**:   
I kind of judge a documentation but the times I have to look up things outside of the actual documentation. In this case it was (almost) none. [fastAPI](https://fastapi.tiangolo.com/) has an outstanding documentation with a great [tutorial section](https://fastapi.tiangolo.com/tutorial/intro/) that gets you started in no-time. Once you got started, it holds a section for everything you would love to do a deep-dive on.

- **it's fast**:   
It's fast to code, and [fast](https://www.techempower.com/benchmarks/#section=test&runid=7464e520-0dc2-473d-bd34-dbdfd7e85911&hw=ph&test=query&l=zijzen-7). Not that I will come anywhere close to fully notice it, but it is good to use something that has the potential to compete with APIs written with **Go** or **nodeJS**.

- **typing**:   
fastAPI uses [pydantic](https://github.com/samuelcolvin/pydantic/) for data validation, so all I do is declade input and/or output models and fastAPI validates the data, converts the output to the type declaration of the models, adds a JSON schema in the OpenAPI path operation and uses the models as automatic documentating system. In case of the output ([response_model](https://fastapi.tiangolo.com/tutorial/response-model/)) fastAPI will also make sure to limit the output data to what is declared in the model.

- **automatic interactive documentation**:  
Writing a documentation for your API is a no-brainer. In Flask I would build my API and at the very least document the endpoints my API provides, the example input parameters and some example output that is to be expected. Now, that's two steps at least - build, document.
With fastAPI I build my API and the documentation is build automatically from the routes, python typing and the input/output models you define. All of this comes with the interactive swagger UI that you can only love.

Now that's my two cents on fastAPI, just give it a try in your next project and follow in the footsteps of other [adopters](https://fastapi.tiangolo.com/#opinions).


