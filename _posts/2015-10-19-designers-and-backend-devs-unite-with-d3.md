---
layout: post
title: "Designers and backend Devs Unite (With D3)"
description: ""
category: 
tags: []
---
You want to do cool things with D3, because you have cool data and D3 makes cool visualizations but D3 is hard. Luckily it's been around for a while and some cool tools have popped up around it to make our lives much easier.    
This post will give a gentle introduction to dc.js, a library for visualzing high dimensional data, and one that is easy(er) to use. In a nutshell, dc.js lets you turn this (ADD IMAGE OF CSV) into this (ADD interactive chart). In this post I'll show you how.

# Some terms
d3 stands for Data Driven Documents, you'll notice that name sais nothing about visualizations - because d3 lets you build "documents" from "data", if your document contains visualzations is entirely up to you. 
But you're reading this because you want to turn your ugly data into pretty pictures so lets clear out a few basic terms and get rolling. 

You have a flat data file, god knows where it came from. Maybe when of the people in BI gave you a csv dump from the database, maybe your boss gave you an excel file with lots of rows and columns, maybe you've been logging what you ate every day in google docs and really want to explore what you're up to.

