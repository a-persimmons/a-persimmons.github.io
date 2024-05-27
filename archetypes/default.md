---
author: ["柿子"]
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
date: {{ .Date }}
summary: ""
tags: [""]
typora-root-url: ../../static
typora-copy-images-to: ../../static/images
categories: ["{{ trim (replace .File.Dir "posts/" "") "/" }}"]
---
