---
title: Building a Social Network in 24 Hours
published: true
---

<figure>
<img class="no-top" src="/social-media-clone/banner.png" alt="installing nginx in ubuntu">
</figure>

NOTE: The first two sections discuss the project on a higher level and the last few cover the lower level systems stuff.

## Intro and Background

Over the course of the last few years, I've found that my increased exposure to higher-level codebases and programs has created a slight obsession with doing things 'the right way'. What I mean is stuff like setting up extensive project boilerplate code, defining extensive types, and spending hours and hours of planning architecture before even writing code.

This is absolutely the approach one should take when working on a serious project; standards are there for a reason after all. But 90% of my projects aren't serious, and I find that I get bogged down at the early stages instead of just focusing on building, especially if the project is just for fun.

To that end, I decided to put other projects on hold and just build something for fun as fast as I could. I wanted to do something with databases and complex queries. In my Algorithms class, we talked about how Google ranks search results. I decided to combine these two things by building a social media service like Instagram from scratch, with a focus on developing an algorithm that builds a custom feed for a user in an efficient manner.

<figure>
<img class="no-top" src="/social-media-clone/insta_arch.png" alt="installing nginx in ubuntu">
</figure>

## Architecture

I know I said this wasn't important, but I do think it's important to detail what I actually ended up building out. I did this as I built the project out, so it got a bit messy in the end. The stack includes NodeJS for server-side Typescript, MongoDB for storing application data, and Firebase for user authentication and storage.

## Goals

Instead of planning out architecture, I spend 5 minutes writing down what I wanted to get done in 24 hours from a high level overview. I settles on a few things. The first was accounts, which need to have the ability to follow and unfollow. Second was posts, which need to have likes, comments, and a caption,. These must be abstracted out into their own models. The last requirement was a feed builder which uses a custom algorithm to build a user's feed.

And some extra stuff if I have time: stories, profile photo editing, profile security, privacy features.

## User Interface

I chose to use NextJS for the UI. I've been using vanilla React for the last 2 years but I committed to only using Next starting with my Markdown Editor project and then with the upcoming rebuild of Fragment. I'm still getting used to it but the server-side rendering is a such factor for me. The application is built as a one page app. This way I could avoid wasting time by using any kind of routing. I took a new approach to building the UI this time by giving all elements debug background colors. I didn't want to waste time with CSS (even though I'm also using ChakraUI) so seeing container bounds for each object can help fix mistakes much faster.

<figure>
<img class="no-top" src="/social-media-clone/colors.png" alt="installing nginx in ubuntu">
</figure>

I decided to split the UI into a sidebar and a main feed component with the sidebar on the right. As far as I remember that's what the Instagram side had (I just realized I never checked). While I didn't spend much time on the UI feel, I did want it to resemble the instagram mobile experience. The result was a cards based system that I eventually linked together using `framer-motion` (I did it anyway). Here's the final look:

<figure>
<img class="no-top" src="/social-media-clone/sitepng.png" alt="installing nginx in ubuntu">
</figure>

The important thing here is that the UI is basically just a shell. Now that I'm looking back at the code, there is absolutely no business logic on the front-end, just an API to communicate with server endpoints.

### Session Authentication

While I know this isn't a necessary part in building the project locally. I did want to ultimately host this online and have people load test it. This means that I had to authenticate the API so that external sources could access data as well as so users couldn't access other user's data.

I decided to use `jsonwebtokens` in order to authenticate sessions. When the user logs in, we return an authentication token that it stores in the browser. This token can then be used to authenticate further requests to other areas of the API and expires after 24 hours, ensuring that stale sessions do not occur.

## Backend

This was probably the most fun part of the project because this is usually where planning is most important and of course, there was no planning happening this time. This was the gateway:

<figure>
<img class="no-top" src="/social-media-clone/code1.png" alt="installing nginx in ubuntu">
</figure>

### Database and Authentication

The auth service was done using Firebase. I've been using Firebase for my applications since I was 12. 8 years later, I haven't found a better solution for quick authentication and storage (although I have moved on from their database service). Because the data for the application resides on MongoDB, we need a way to link the Firebase user with their data in the MongoDB. To do this we can use the Firebase user's `uid` field as the `id` parameter in the MongoDB schema for a user. This helps more easily query data in the future.

There were a couple of required models for this database: Account, Post, and Comment. Each one of these had id's linked to a user or post, respectively. Something I learned by the end of the project that I didn't know before was just how often production databases use dirty duplication in order to save processing. While I knew this was faster, I just assumed it was bad practice. In order to scale this service, however, this is one of the first places I would start to optimize.

The main reason I chose MongoDB over Firebase was simply due to the ability to generate complex queries and indexes on your data. I also loved the ability to set my own schemas as individual parts of a larger NoSQL database. This makes adding new fields and relating them to other objects becomes easy. However, it really shines when it comes to making queries. For example, getting comments for a user's post is as simple as defining a query like:

<figure>
<img class="no-top" src="/social-media-clone/code2.png" alt="installing nginx in ubuntu">
</figure>

### Feed Building and Aggregations

Building a user's feed was one of the most important challenges to tackle in this project. The challenge wasn't figuring out what posts to show a user, but rather how to pull the data efficiently. The algorithm will be the old basic Instagram one: display posts by accounts that a given user follows sorted by chronological order. This is easy conceptually, but is an inefficient query if done improperly. Logically the first step is to query for a list of users and list each post per user. Next, sort the list of posts by timestamp. Finally, return the first `x` posts

MongoDB offers two ways to do this efficiently without using queries: Indexes and Aggregations. I chose to use aggregations because they were a bit more dynamic and I wasn't working with a huge amount of data. If I was looking to scale, I would also look here. The final feed algorithm was this aggregation:

<figure>
<img class="no-top" src="/social-media-clone/code3.png" alt="installing nginx in ubuntu">
</figure>

## Summary

I think this project helped reinforce a lot of what I've been learning systems-wise over the last couple months. While this project was a fun escape from more serious projects, it was nice to see that some 'good' practices had already become habits while it was interesting to see where I tended to cut corners. It could have been much easier to do this with Javascript instead of Typescript and just use `any` types, but there is something rewarding in building something modular and extendable.

I do want to load test this project so I will work on making it ready for public use in the next few days. I specifically hope to explore reducing the amount of queries made per user session as I don't believe I built with full efficiency in mind.

[Check out the social media site here](https://social.micahelias.com)

<iframe width="560" height="315" src="https://www.youtube.com/embed/KUmS5fgMvmY" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
