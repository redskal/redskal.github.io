---
title: "Playing with Smishing"
layout: post
excerpt_separator: <!--more-->
category: Blog
---

To paraphrase the biblical texts, "give a man a credential, he'll pwn for a day; teach a man to phish, he'll pwn for the rest of his life." So let's look at setting up smishing infrastructure with [Vonage](https://www.vonage.co.uk/).

<!--more-->

Why Vonage? Because they're UK-focused, as is my work. There were other tools and techniques I found through Google, but services such as fast2sms appear to require a +91 phone number to register an account. Vonage requires far less leg work.

Sending an SMS message to my personal phone in order to test the service cost roughly €0.04. I haven't shopped around as Vonage met my requirements and I don't actively run smishing campaigns.

The requirements I had for a service were:
- Custom sender ID
- Not bank-breaking
- An API I can build tooling around.

Vonage allows you to use numeric or alphanumeric values for the sender ID. The interesting one for this scenario is alphanumeric values. The rules for the sender's custom caller ID as an alphanumeric value were:
- Must be a string of up to 11 supported characters
- Supported charset is `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789`
- Cannot contain spaces

The cost didn't make my wallet weep, so that seemed reasonable. Vonage provide a simple API to send SMS messages with several SDKs available; unfortunately, no Golang SDK is offered. However, there are C#, Java and Python options. Node and Ruby can get in the bin and be set aflame.

After a little trial and error, things were being delivered. Some ironing out of kinks through cURL-based debugging got the job done:

<img src="/img/curl-request.png" class="article-image">

The message was received after a couple of minutes. Hopefully paid-for credits would cause the annoying appendix to be removed, but it's working:

<img src="/img/knowbe4phish.jpg" class="article-image">

The Vonage Dashboard gives some nice statistics for your campaign regarding messages delivered and rejected which might help in fine-tuning:

<img src="/img/vonage-dashboard-stats.png" class="article-image">

To conclude, it seems as though Vonage is a viable option for operating smishing campaigns for red teams (at least in the UK). Decent documentation[^doc-ref] of their fairly simple API means that developing tooling will be light work, which is always a bonus.

[^doc-ref]: Vonage: Send an SMS: [https://developer.vonage.com/en/api/sms#send-an-sms](https://developer.vonage.com/en/api/sms#send-an-sms)