---
author: "Abhay"
title: Sessions - Backend Basics [1]
description: "HTTP is a stateless protocol, but making applications stateless from users perspective will make them login again and again hell. To abscract away this stateless-ness we use sessions."
date: "2021-07-19T14:23:34+00:00"
template: "post"
draft: true
slug: "backend-basics-1"
series:
  - backend-basics
tags:
  - "backend"
  - "http"
  - "cookies"
hidemeta: false
comments: false
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/rathod-sahaab/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

# Who are we?
A crucial part of any personalized application, is knowing who the user is. Pair that with need for security now we need authentication too. How do we even achieve those when the underlying connection we rely on is itself stateless.

## A Refresher of HTTP
