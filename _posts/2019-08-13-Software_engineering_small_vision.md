---
title: Small mind on software engineering
date: 2018-04-03 11:00:00 
---

Recently an old trend brought up in our country - Social network for Vietnamese. The trend is initialised by our [Minister of Information and Communications](https://vnexpress.net/kinh-doanh/ong-nguyen-manh-hung-nen-lam-mang-xa-hoi-cong-cu-tim-kiem-thay-facebook-google-3952971.html). It's funny that after years trying to do the same thing again and again and we must admit that building a local social network is a wrong way, and we still do it again. 

It's not only funny but also a waste of time and resource, follow a wrong trend, many local company will restart old project and put software engineers in to that project, on a dream to build thing users never expect to use. 

The pioneer: 

- https://biztime.com.vn/
- http://vivavietnam.vn/
- many to come

While the world changes every seconds, trying to invent, adapt, learn, master and build new thing, we go back to the old day, build the old thing. Actually we will not build it, we will copy from the other, in this case - Facebook. History proves that only Chinese can do that, their local market is big enough to fight again global market, while our products expect millions of user, they already have billion of internet user, ready to use new product if it's interesting enough. 

> The definition of insanity is doing the same thing over and over again and expecting a different result

But the main subject of this post is not about idiot projects, but about the goal. When starting new project, we expect to be small, not to be great. We limit ourselves in local market, instead of going global. We copy successful project instead of making prototype, design  and invent on our own. 

9 years ago I joined a company. At that time we also built a social network, yes, a social network for Vietnamese. 9 years, 24 years old, joined a realy big company with over 1000 employees, mostly software engineers was a dream come true. I was so excited, a team with less than 50 people working together to build product we believed it will be successful and going big, but it's only big in our dream, we tried to make our product **local**. That social network got some attention, we got nearly 10 million users, mostly teanager and gamer. But it's not enough for user to use a social network, which got nearly the same features as Facebook, less attractive, poorly copy-cat, and especially, it's so local. When Facebook realised Viet Name is really a big market, can generate huge revenue, they put more effort to localize UI, language, and user experience, our social network failed after 2 years. The team was splitted, a half stayed and tried to keep that social network living, the other half moved on to build a more interesting product, another copy-cat product. 

We did not only make our product local, we also made our software local. [There's some comment out there talking about Viet Name companies, calling themselves technology company, without creating or contributing a single line of code for opensource project, especially the fact that most or all of their products use open source project](https://vnhacker.blogspot.com/2019/06/chuyen-bay-gio-moi-ke-made-in-vietnam.html). 

So the question is why? Why we developed a social network, with thousands of component, millions line of code, running on over 300 physical servers, but cannot open source a single component or contribute a single line of code. It's because our software is local. 

We developed software only for our product, a software will stick with many other softwares, and all of its dependencies stick to other dependencies, which also developed for our product, it's not designed to be common-used, and not designed to be opensourced. 


![small mind](/images/small_mind.png)

Saying something on Facebook is super easy, actually doing something you've said on Facebook is another story. 
A social network which had 10 millions of user is going to be opensourced is a great story, but the question is how can you opensource it? and how people can actually use it. 

The time when we developed that social network, thousands of micro services was deployed and running. At first, it was written in PHP, customised Zend Framework, then converted to Java and C++, every single component at once. A feature composed of many software, for example: Feed has feedStorage, FeedService which connect to ProfileService and FeedStorage, FeedCaching, FriendService, PhotoService,...., not talking about other services like: ZooKeeper, Kafka, redis... it means to opensource Feed, you have to opensource all of it dependencies, and dependency has dependency, you have to opensource everything. 

![dependencies hell](http://i.imgur.com/fanTVix.png)

And it's not enough. There's not a single person of the company can deploy the whole product, not a single person. That's why as time went by, [many features is disabled or unusable](http://genk.vn/net/phan-2-mang-xa-hoi-zing-me-niem-tu-hao-nay-chi-con-la-thanh-pho-ma-20140330223401903.chn) 

> **Why opensource something no one can use?**


We have small vision in product road map, small vision in product developing, and all of that because we also have small mind. Time to time when solving a problem we do it the ad-hoc way, adding some hack and think it's cool and clever, things end up with plenty of hack, easy to forget and unmanagable. 

When using opensource software, we downloaded from github, follow step by step installation from `README`, make some request, collect some response, connect it to our software and bring it to production. When it does not as work as expected, we blame the guy opensource it, switching to another opensource software, stuck in that loop until an awesome thing come up. That's the way we developed software. 

When planning for new feature or new product, instead of sitting down together, making dicusssion to findout the solution, we always try to find an example: what did Facebook do, what did Netflix do... and try to make a copy of public solution, with not enough knowledge and experience. 

We can not go big because at the end, our product is made up by thousands of POC, plenty of problems cannot be solved. 

