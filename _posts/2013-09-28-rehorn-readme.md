---
layout: post
title:  "RedHorn: readme"
date:   2013-09-28 18:52:24
---
RedHorn
=======
This is implementation of transport layer inspired by Andrei Alexandrescu's Modern C++ Design. 
RedHorn isn't intended to be fully functional library which ready to use as part of comercial
or non-comercial project. It's more about demonstration of technics and ideas which can be
used to develop transport layer which match requirements of specific project.

## RedHorn's goals
* Flexible and extensible transport layer.
* Changes in packet structure should not cause malfunction or require changes in send/recv code, 
required code changes should be minimal.
* C++ struct syntax should be preserved.
* Support of communication between C++ and Python clients.
* Single packets definition for clients written on C++ and Python.

## Requirements:
* g++ 4.7
* python 2.7
 
