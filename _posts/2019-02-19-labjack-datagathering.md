---
layout: post
title:  "LabJack LJMs and automated data collection"
date:   2019-02-19
excerpt: "Determining torsion constants is easy if you work really hard for it."
project: true
tag:
- jekyll 
- moon
- blog
- about
- theme
---

For the last few months, I've been working with one of my former professors to try to develop a method of collecting data from a [magnetically dampened torsion pendulum][0] in some kind of reliable and scalable way. We've been using [Labjack T7 Pros][1] due to their versatility, scalability, and open-source libraries; this arrangement has only been hampered by the fact that developing scalable data collection functionality requires a lot of work from the ground up.

Effectively, our needs were as follows:

1. Easily or automatically connect to a T7 and handle stream opening/closing nuances
2. Given the Labjack device, network connection, and hosting computer, determine the maximum frequency of data scans, and how many scans should be fit into a packet.
3. Catch and recover from any kind of interruption or error.
4. Allow the recorded data to be accessible for realtime backup and analysis.





[0]: http://www.paulnakroshis.net/blog/
[1]: https://labjack.com/products/t7