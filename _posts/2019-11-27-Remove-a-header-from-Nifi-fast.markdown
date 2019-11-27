---
layout: post
title:  "Remove a header from Nfi fast"
date:   2019-06-09 08:27:29 -0400
categories: NiFi
author: Thomas Kreutzer
---

### Efficient header removal from a file in Nifi:

This one is fairly simple, use sed and the ExecuteStreamCommand processor. 

## Configuration

Command Arguments: 1d
Command Path: sed
Ignore STDIN: false
Working Directory: No value set
Argument Delimiter: ;
Output Destination Attribute: No value set
Max Attribute Length: 256