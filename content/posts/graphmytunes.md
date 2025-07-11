---
title: 'GraphMyTunes: a new tool for visualizing your Apple Music library'
date: 2025-07-06T06:00:00-0700
description: Users of Apple Music (formerly iTunes) may be interested in my new Python-powered tool that generates insightful graphs and visualizations of your music library.
slug: graphmytunes
tags:
    - python
    - music
    - itunes
---

A significant portion of my musical identity is contained in the metadata of my Apple Music library, which I’ve lovingly curated for more than 20 years. I’ve always been interested in using this metadata to learn more about my own musical interests and listening habits.

Some simple metadata-driven insights are available in the Music app itself, if you know where to look. However, the data I especially enjoy is more complex than the Music app provides, and often requires the combination or aggregation of multiple metrics. Some examples of particularly interesting questions are:

- **What album or artist have I listened to most?**
- **Which albums or artists do I skip the most?**
- **Which genres have the highest average rating?**
- **How much time have I spent listening to specific genres?**
- **Which decades of music are most represented in my library?**

So I wrote a Python tool to answer those and other questions, and hopefully help other music lovers do the same!

[![GraphMyTunes Logo](../images/graphmytunes-logo.png)](https://github.com/homebysix/GraphMyTunes)

{{% mark %}}GraphMyTunes is my new open source tool powered by Python and Matplotlib that analyzes your Apple Music library and produces interesting graphs.{{% /mark %}} I've published the project on GitHub at https://github.com/homebysix/GraphMyTunes.

## How do I use it?

To get started using GraphMyTunes, use pip to install it in your Python environment of choice, then provide an XML file you exported from the Music app.

    pip install GraphMyTunes
    graphmytunes ~/Music/LibraryExport.xml

See the [Quick Start](https://github.com/homebysix/GraphMyTunes/?tab=readme-ov-file#quick-start) section of the README for full step-by-step instructions.

## What can it show me?

I've included 40+ analysis modules in GraphMyTunes, which output visualizations including:

### <span style="font-size: 2rem;">📊</span> By play count

Beyond just the top played songs, GraphMyTunes can show you your favorite albums, artists, or genres by play count.

![](../images/graphmytunes-artist_plays.png) ![](../images/graphmytunes-genre_plays.png)

### <span style="font-size: 2rem;">⏱️</span> By total play time

{{% mark %}}But simple play counts don't truly reflect the most important thing you've invested in your music: your time!{{% /mark %}} So GraphMyTunes can also show you which albums, artists, and genres you've spent the most minutes of your life listening to.

![](../images/graphmytunes-album_playtime.png) ![](../images/graphmytunes-genre_playtime.png)

### <span style="font-size: 2rem;">⭐️</span> By average rating

I enjoy using star ratings to classify how I feel about memorable songs. GraphMyTunes aggregates your ratings across albums, artists, and genres to give you insight into which ones are highest-rated on average. (These are some of my favorite graphs, as I think they reveal a less tangible sentiment than play count.)

![](../images/graphmytunes-album_avg_rating.png) ![](../images/graphmytunes-artist_avg_rating.png)

### <span style="font-size: 2rem;">🧑‍🎤</span> By decade

If the release year of the music in your library is accurate, GraphMyTunes lets you explore which decades of music are most or least represented — by track count, play count, or total play time.

![](../images/graphmytunes-decade_plays.png) ![](../images/graphmytunes-decade_playtime.png)

### <span style="font-size: 2rem;">🔂</span> By average daily plays

Another interesting metric surfaced by GraphMyTunes is the average plays per day between the time you added a song to your library and today. This can also be broken down by album or artist.

![](../images/graphmytunes-album_avg_daily_plays.png) ![](../images/graphmytunes-artist_avg_daily_plays.png)

### <span style="font-size: 2rem;">🎼</span> ...and many more!

I've included histograms that break down your library by file kind, sample rate, beats per minute, date added, or date last played. There's an overview chart that shows you basic stats about your library including its age, size, and total listening time. GraphMyTunes even produces CSVs that reveal your personal top hits by year, if you're feeling nostalgic.

## What are the limitations?

The available metadata includes date added, play count, and last played. However, there's no comprehensive log of every date/time a given song was played, which makes it difficult to answer questions like "What are some songs I used to listen to a lot but haven't lately?" or "Which days of the week do I listen to the most music?" (We can estimate answers to those questions using last played information, but it's not as accurate as a full play log would be.)

I'm also focusing on Apple Music support at this time, so you'll have to look elsewhere if you use a different music service or app.

## How do I add more graphs?

I've designed GraphMyTunes to be modular and expandable to make it as easy as possible to add more interesting visualizations in the future. If you're familiar with using Python and matplotlib for statistical analysis, I'd love your help! See the [contribution guide](https://github.com/homebysix/GraphMyTunes/blob/main/CONTRIBUTING.md) to get started.

---

## A fresh perspective

I hope you enjoy using GraphMyTunes as much as I've enjoyed creating it. My goal is to reveal musical tastes from a fresh perspective — and perhaps reveal something insightful about the listener as well!
