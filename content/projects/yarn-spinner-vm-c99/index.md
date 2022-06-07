---
title: "Yarn Spinner C99 Runtime"
date: 2022-04-20T11:15:50+09:00
kind: projects
techstack: C99
description: work-in-progress, easy to integrate C99 port of Yarn Spinner's VM runtime.
draft: false
---

# Yarn Spinner C99 Runtime

![project banner](./banner.png)
[Github Link](https://github.com/komugi1211s/yarn-c99)

## About

C99 port of [Yarn Spinner](https://yarnspinner.dev/)'s Runtime, a tool for writing game dialogue.  
written for integrating into my own game development.  
**this project is not associated with the owner/developer of Yarn Spinner.**

### Feature
 - API contained in a single header, which makes it easy to integrate. (although currently depends on protobuf, which is unavoidable.)
 - reads a set of `yarnc` + `CSV` files.
 - register C functions into Yarn Spinner easily, done in a way similar to Lua.
 - original implementation of hashmap, usable in user space.
 - original implementation of memory allocator, usable in user space.
