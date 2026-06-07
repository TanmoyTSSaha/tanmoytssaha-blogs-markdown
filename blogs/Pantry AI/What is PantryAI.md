---
title: What is PantryAI?
slug: what-is-pantry-ai
date: 2026-05-03
author: Tanmoy Saha
tags:
  - series
  - pantryai
  - learnings
  - programming
  - golang
  - flutter
description: Kicking off a new product I'm trying to build in public within next 100 days.
reading_time: 10
draft: true
series: Pantry AI
series_slug: pantry-ai
series_order: 1
---

# What is Pantry AI?
As a software engineer living alone in a city far from my hometown, I struggle with one thing more than any sprint deadline — managing my pantry. Every evening I open the fridge and ask myself "what do I cook today that's actually healthy?" — only to discover vegetables I forgot about, now past their best. I know I'm not alone in this.

According to the [Food Waste Index Report 2024](https://wri-india.org/perspectives/india-must-reduce-food-waste-source), Indians waste 55 kg of food per year per person. A big driver of this is simply forgetting what we have, buying duplicates, and having no plan. For students, solo professionals, and urban families juggling busy lives, pantry management is one of those invisible problems that quietly drains both money and nutrition.

So I'm building **PantryAI** — an AI-powered application to help you manage your pantry smarter, plan meals around what you already have, and optimise your grocery shopping. This series documents the entire build in public, from idea to launch, within the next 100 days.

## What is this application about?
- Creating weekly customisable meal plan: This app will have features to plan and customise weekly meal plan according to your pantry and your nutrition needs.
- Managing the grocery: It will help you manage your grocery, plan meals using the grocery products before expiry.
- Suggesting grocery shopping plan based on weekly meal plan: Suggestion to buy product from quick commerce applications for the planned meals. Suggesting better alternatives of products being used.
- Distributing grocery shopping carts: planning and optimising shopping carts among quick commerce application to get lowest price. It will help you saving some extra money.
- Showing product's nutrition values: By scanning product bar code this application can give you nutrition values of that product.

## What technology stack will I be using for this application?
- Frontend: Flutter
- Backend: Golang
- Database - Postgres, Redis
- Authentication - Firebase

Let's understand why I have chosen the above technologies for the application stack. 

Starting with frontend I have chosen flutter for this application. Because using flutter I can develop application for both android and iOS using the same code. And in future I can expand the codebase for web application and windows and other os. So till now we have eliminated Swift, Java, Kotlin and objective-c. As these languages are for platform specific development. Now as I'm little less familiar with javascript environment I have not chosen ReactNative or ionic. Maybe I can give it a try once I learn JavaScript properly.

For backend I have chosen golang. Because I'm learning golang recently and I want to build some real-world application while learning it. And we have built-in concurrency in golang which we can use during the development. 

For storing data we will be using postgres as it can handle most of the things. And for caching data we will be using redis.

For authenticating users we will be using firebase auth as it will save my effort in authentication part.

This is **Part 1** of the **PantryAI** series — the idea, the problem, and the tech stack.
Here's what's coming next:
- **Part 2** — System Design of the Backend: How we'll architect the Golang backend, database schema, and API design.
- **Part 3** — System Design of the Frontend: Flutter app architecture, state management, and navigation structure.
- **Part 4** — UI Design of the Application: Design system, wireframes, and screen-by-screen breakdown.

If you're building something similar, struggling with food waste, or just curious about how a solo developer ships a full-stack AI app in 100 days — follow along. I'll be posting every part of this journey honestly, including the mistakes.
Drop your thoughts in the comments below. I'd love to know: do you struggle with pantry management too?