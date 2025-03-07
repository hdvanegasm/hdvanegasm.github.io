---
title: "My experience with Arch Linux"
date: 2025-02-11
draft: false
math: true
editPost:
  URL: "https://github.com/hdvanegasm/hdvanegasm.github.io/tree/master/content"
  Text: "Suggest Changes"
  appendFilePath: true
---

This is a rolling blog post about my experience with Arch Linux. This content may help somebody.

# Mar. 7 - 2025

I have found an error in the package `dwarffortress`. I installed the package using pacman as always. However, when I executed the program, I got the following error:

```
/usr/bin/dwarffortress: line 15: 13472 Segmentation fault      (core dumped) LD_LIBRARY_PATH="${LD_LIBRARY_PATH:+$LD_LIBRARY_PATH:}${DF_DIR}" $DF_DIR/dwarfort "$@"
```
After printing the output of `journalctl -r`, I got the following output:

```
Module [dso] without build-id.
```

After that, I recently did a major update to the system and the error was completely solved. I still don't know what happened and how could I solve the problem at that time. 


# Feb. 11 - 2025

I have found Arch Linux very stable so far. The installation comes with known quirks, but their excellent documentation covers almost any question I have. I found out that the GNOME desktop environment is not that great, or at least it does not feel as snappy as the KDE desktop. Hence, I chose KDE as my main desktop environment. 

The only problem I have faced so far is that Visual Studio Code has a minimal delay while writing. I don't know why that happens, but once I find a fix, I will post it here. The delay does not bother me a lot because it's very small.  

