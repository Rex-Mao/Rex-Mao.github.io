---
title: Charset & Encoding
date: 2022-06-17 15:28:46
tags:
---

## Charset
Charset is the set of characters. For example, GB2312(charset) has contained basic simplified chinese characters; Unicode(charset) has contained 144697 characters of 22 languages.

## Encoding
Encodings map the character and the binary code representation. It depends the way these characters are stored into memory. When u want to transport some bytes, it should be encode and decode in the same encodings, otherwise u will get wrong results. 

```
>>> text = u'您好'

>>> text_bytes_8 = text.encode('utf-8')
>>> text_bytes_8
b'\xe6\x82\xa8\xe5\xa5\xbd'
# 2 hex-based [2^4 * 2^4] bits can be represent by 8 binary-based bits [2^8]
# \xe6 [ e6 = 14 * 16^1 + 6 * 16^0 = 230 = 1110 0110]
>>> text_bytes_8.decode('utf-8')
'您好'
>>> text_bytes_8.decode('utf-16')
'苦\ue5a8붥'

>>> text_bytes_16 = text.encode('utf-16')
>>> text_bytes_16
b'\xff\xfe\xa8`}Y'
>>> text_bytes_16.decode('utf-16')
'您好'
>>> text_bytes_16.decode('utf-8')
Traceback (most recent call last):
  File "<input>", line 1, in <module>
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xff in position 0: invalid start byte
```

## Relationship
Every charset maybe contain one or more encodings, like Unicode. Unicode charset has utf-8, utf-16, ucs-2 encodings.