---
title: Charset & Encoding
date: 2022-06-17 15:28:46
tags:
---

# Charset & Encoding

## Charset
Charset is the set of characters. For example, GB2312(charset) has contained basic simplified chinese characters; Unicode(charset) has contained 144697 characters of 22 languages.

## Encoding
Encodings map the character and the binary code representation. It depends the way these characters are stored into memory. When u want to transport some bytes, it should be decode and encode in the same encodings, otherwise u will get wrong results. 

## Relationship
Every charset maybe contain one and more encodings, like Unicode. Unicode charset has utf-8, utf-16, ucs-2 encodings.