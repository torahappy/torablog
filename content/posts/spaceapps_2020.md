---
title: "COVID-19 Space Apps Challenge やってみた"
tags: ["SpaceAppsChallenge", "COVID-19", "つき", "Moon"]
date: 2020-06-15T20:10:53+09:00
---

月面をGoogle Mapみたいにするプロジェクトをやってみた。

Demo: <https://aidatorajiro.github.io/spec/>

↓えいご　の　せつめい↓

# Walking on the Moon - A Moon Mapping Project

## Abstract

The moon looks like a metropolis. There are so many artificial structures on the moon. This project uses AI technology to automatically detect structures (such as roads or buildings) from the surface picture of the moon. Names for detected buildings are also generated (like University, Station, etc.). The project provides a web app in which you can enjoy exploring building info and plan your journey or hiking in the moon, like Google Map. The entire city map of the moon is now here....

## Motivation

Now, we are having a difficult time. Not only COVID-19. Political conflicts and financial crisis are also coming. In such a time, imagination is very important thing, because you can fulfill your curiosity! It can also help you to come up with new idea, which is so important when the future is unclear. Maps are so good when you imagine about elsewhere. Just seeing it and recognizing what things there are, you feel as if you were in the place you see.

## Method

I used an open source project (https://github.com/mitmul/ssai-cnn) to detect objects (roads and buildings). I also used many open source software such as GIMP and Inkscape to process, transform and translate data. Of course, many Python and Javascript codes are written.
The flow of data processing is as follows:
(1) CGI Moon Kit https://svs.gsfc.nasa.gov/4720 is the original dataset. I used 8K (8192 × 4096) version of it.
(2) Use https://github.com/mitmul/ssai-cnn to detect object from CGI Moon Kit. As it is an old library, most of requirement libraries of it are outdated. Outdated libraries does not provide binary form of build, so I upgraded Chainer, which was one of the requirement, to 1.7.10. Chainer v2 or later is incompatible with the library, so 1.7.10 is the best. The output of the software is formatted in RGB PNG file, in which R component denotes roads and G component denotes buildings. For 8K version, it took roughly 0.9hour to get result using Intel i7 CPU.
(3) For building detection, I used GIMP to extract G component of the image. Then I used "threshold" filter to get black-or-white colored detection result. After that, Inkscape helped me convert the image to svg format, which contains group of graphs. I wrote a SVG parser that converted it to the list of building positions by calculating the center of gravity for each graph.
(4) For road detection, same as (3), I extracted R component, used "threshold" filter and vectrized the result using Inkscape. The group of graphs were then converted to SciPy's csr matrix, because graphs can be represented as a sparse matrix. The csr matrix were used to calculate shortest path between two points, which is one of the functionalities in the website.
(5) Also, the road graphs were isolated each other, so I forced each point to connect its neighbors. The road graph processed in (4) and (5) were then re-converted to SVG, which is one of the backgrounds in the website.
(6) The web app consists of two elements: server side and client side. The server side is written in Python and the client side is written in Javascript and Vue.js.
(7) The server side possess all graph information (roughly 30MB). It can do two calculations: getting shortest path and getting nearest node.
(8) When the client access "/get_shortest_path/A/B" via GET request, the server returns every points (a pair of node id and node position) which the shortest path between A and B contains. Note that A and B are node ids. Also, node position is the position written in the original SVG, so it is not longitude or latitude.
(9) When the client access "/get_nearest_node/X/Y", the server returns the nearest point from the position (X, Y)
(10) The client possess resized version of original dataset (https://svs.gsfc.nasa.gov/4720), showing it in the background.

## References

Web service : https://aidatorajiro.github.io/spec/ (Japanese title つきをあるいてみる)

CGI Moon Kit https://svs.gsfc.nasa.gov/4720

#artificial intelligence, #map, #road detection, #building detection, #imagination
