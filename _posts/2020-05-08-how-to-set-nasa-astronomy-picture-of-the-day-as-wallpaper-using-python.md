---
title: How To Set NASA Astronomy Picture Of the Day As Wallpaper Using Python
layout: post
author: Sichong Peng
date: '2019-05-08'
slug: how-to-set-nasa-astronomy-picture-of-the-day-as-wallpaper-using-python
excerpt_separator: <!--more-->
categories:
  - Python
tags:
  - Python
  - Automation
---

NASA uploads a selected astronomy picture daily as their [Astronomy Picture of the Day](https://apod.nasa.gov/apod/astropix.html) (APOD). For a space sci-fi junkie myself, I find these pictures extremely pretty and I want to use them as my wallpapers! But obviously downloading them from NASA everyday is a huge hassle. Luckily, NASA provides open APIs that we can make use of to automate this process.
<!--more-->
The NASA open APIs can be found at https://api.nasa.gov/. To use this API, you'll need an API key. You can generate your own key on their website. But if you're only using this API to get their APOD once a day, you'll be perfectly fine with their demo key (DEMO_KEY). For the rest of this article, I'll be using DEMO_KEY to access NASA open database.

To access NASA APOD in python, we use a module called `requests` to make web queries:
```{python}
import requests
metadata = requests.get('https://api.nasa.gov/planetary/apod?api_key=DEMO_KEY')
```

`metadata` contains metadata information of the APOD. We can use `metadata.json()` to retrieve a Json dictionary of these data:

```{python}
print(metadata.json()['explanation'])
```
Above statement will print out caption for the APOD. To download the picture, we use `metadata.json()['hdurl']`:

```{python}
import urllib
file_path, header = urllib.request.urlretrieve(metadata.json()['hdurl'], filepath)
```
Where `filepath` is the path to store the picture file.