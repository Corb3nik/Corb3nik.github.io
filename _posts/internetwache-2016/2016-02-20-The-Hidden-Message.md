---
layout: post
title: The Hidden Message (misc50)
category: internetwache-2016
---

## Description
My friend really can't remember passwords. So he uses some kind of obfuscation. Can you restore the plaintext?

---

## The Challenge

Opening the challenge files, we obtain a readme with the following content:

    0000000 126 062 126 163 142 103 102 153 142 062 065 154 111 121 157 113
    0000020 122 155 170 150 132 172 157 147 123 126 144 067 124 152 102 146
    0000040 115 107 065 154 130 062 116 150 142 154 071 172 144 104 102 167
    0000060 130 063 153 167 144 130 060 113 012
    0000071

Looks like a hex dump, but the values are three digits long, looks like octal.

Convert the values to ascii, and you obtain this base64 string :

    V2VsbCBkb25lIQoK
    RmxhZzogSVd7TjBf
    MG5lX2Nhbl9zdDBw
    X3kwdX0K

Decode it and obtain the flag : IW{N0_0ne_can_st0p_y0u}




