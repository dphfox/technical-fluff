---
layout: post
title:  "Two cyber-social freedoms"
image:
  path: "/assets/posts/two-cyber-social-freedoms/thumb.jpg"
  width: 256
  height: 256
comments_url: ""
---

There are two cyber-social freedoms: the freedom from platforms, and the freedom
from your friends' platforms.

We already have freedom from platforms, because the web is fundamentally free.
Do you hate Twitter? There's a whole host of other platforms out there you can
go and choose from! Don't want to go to a platform? You can easily host your own
using various blogging software or site generators, even for free. I made this
entire blog with nothing but my own two hands and a browser tab open to Google
basically every step of the way. If you want to be free from platforms, you can
do it. It doesn't have to cost you money and you don't have to manage hardware.

The problem is that other freedom. You are not free from your social graph. You
may be free from Twitter, but are you free from your friends that use Twitter?
The answer is - unless you have no friends or no interest in them - you are not
free at all. To investigate the consequences of this, we need to talk about the
internet - trust me, I promise it's related.

-----

The internet gives you the ability to send binary data to any computer that is
connected to it. This is not useful on its own, because that binary data could
mean literally anything.

To make it useful, we invent standards for how you can arrange that binary data
to convey different messages and concepts. Then, when internet users send binary
data between each other, they can meaningfully communicate so long as both of
them understand that standard.

This is where the original vision for the internet comes from. If we can define
these common standards, then we can make the world's computers cooperate to
achieve all sorts of social interconnectedness! You can send messages to your
mum in Singapore instantly, all from the comfort of your Icelandic nook. You can
browse online businesses to find that *perfect* cushion cover, and after sending
some data to your online bank to transfer some money, that business can send it
to your doorstep.

It sounds wonderful, but there's always been a problem with this vision -
hardware. If you want the world to be able to talk to you through your computer,
you're going to need to get set up to accept communications from the whole wide
world, and you're going to need to be connected constantly just in case someone
tries to talk to you when you're asleep. Not to mention you need to make sure
not to open yourself up *completely* - don't want people accessing your printer,
your security cameras or even your laptop, right?

It's pretty obvious that managing your own computer hardware was not going to
catch on like that. As the internet developed, we saw the responsibility of
managing server hardware get quickly off-loaded to other people, leaving us to
merely use and direct those servers. Simultaneously, not everyone is interested
in programming servers. Many people just wanted to have a server that sends out
web pages with blog stuff on them, for example, so for that purpose you'd find
various software that manages those blog internals for you, and exposes a more
friendly UI for you to do things like 'writing blog posts'. At this point, all
you'd need to do is buy yourself a server in 'the cloud' and install your
software on it - once done, you too can have a server that sends and receives
binary data of your chosen kind.

Of course, we'd start seeing the convergence of these things too. Services that
combined servers and software began popping up around the place, so you could
just sign up with them and get a blog, or photo-sharing page, or extra storage
space you can access from anywhere, or whatever. They managed both the hardware
and software on your behalf.

The most extreme convergence happened with messaging and social spaces. The idea
was simple - people don't want to think about things like domain names, servers
or software just to talk with each other online, so why not have a single piece
of software on our own server that just provides a medium for communication
without fuss? People would make accounts via our service, and submit data to our
service. Then, other people on our service can receive that data. All you need
is a connection to the service - no need to set up a server at all.

-----

This is the world we now live in. Facebook did it first, Twitter epitomises it.
You can see how we now do a lot of our social stuff in these centrally-managed
spaces nowadays.

The problem is that these spaces don't talk to each other. You're free to choose
any one of these spaces to sign up to, but your space is never going to link up
with any other space. If you're on Twitter, but your friend is on Cohost, you're
unable to talk to them. That's because Twitter's server doesn't use any standard
for sending and receiving that kind of data, so it's unable to talk to Cohost's
server.

This is where the lock-in comes in. Leaving a certain space becomes a big event.
#deletefacebook. #joinmastodon. You have to make a big deal of your exodus,
because your friends will be cut off from you unless they come along with you
too.

Because most people don't want to be cut off from their friends like that,
simple social inertia then becomes a point of leverage against you - your
*friends* are the ones who control when you leave. This is the 'network effect'
you've probably heard of before; people only join and leave platforms when a
critical mass of their social graph joins and leaves that platform. For some
people, the critical mass is lower - they're the trendsetters. For others, it's
higher - they're the laggards.

It's not just your friends who exercise power over you. The owners of these
social spaces can, too. It doesn't matter whether it's Twitter, Cohost or Hive -
if you own a social space that people are unwilling to leave like that, then you
can get away with making unpopular decisions because you have an audience that
are holding themselves captive. Don't like how many promoted tweets Twitter
shows you? Too bad - they know you're not going to run away to Mastodon, because
your friends won't follow you there unless they fuck up hard enough to cause a
mass exodus, so sit there like an obedient dog and consume their advertising.
What's that? They had a massive data breach revealing personally identifiable
information that puts you directly at risk? Don't worry, you'll have to continue
to depend on their unreliable security in the future. Employees looking through
your DMs and shadowbanning your friends to punish them for speaking out? You're
not going to do anything about it, and they know it. You're docile.

I want to be extra clear here. I'm not here to badmouth your favourite,
good-natured social space. Good social spaces are like good employers; you can
absolutely find wonderful people who treat you well and care about you. I'm
particularly warm to the people at Cohost, who seem to be pretty reasonable and
progressive. But again, much like good employers aren't a defense of systems
that enable bad employers, Cohost should not be used as a defense of walled
gardens that enable social spaces to control their users.

-----

I think it should be clear by this point in the blog what my opinion is. I'll
spell it out here for good measure: central social spaces are not the problem.
The problem is their lock-in characteristics, which give them an undue level of
leverage over people, and so that's what we need to take aim at.

Let's remember what the fundamental reason for this lock-in was: these central
spaces only mediate communication between users of that space. There is no
protocol for these spaces to work together the enable communication between
spaces. The closest thing we have to that kind of cross-connection are bots,
which just mindlessly repost things from one service to another using an
automated user account. That just isn't going to cut it for proper integration.
What we're talking about here is called *federation* - the ability for our
spaces to work together to enable every user inside of them to talk freely with
each other.

So, how do we get these services working together? Remember back to what the
internet is; it's a bunch of servers exchanging binary data which they all agree
to format using some kind of standard. We'll need some kind of standard that
allows for federation, then; if we have a standard like that, we can implement
it as part of our server's software, and now it can federate with any other
server that implements that standard correctly!

You might think that this is a fantasy solution that's never going to happen,
but actually it's already here, and has been for years. You might have heard of
the W3C; they're the guys who standardised HTML, CSS and JavaScript for the web.
It turns out they also published a standard called ActivityPub in 2018, which
allows for servers to federate together and provide exactly the social functions
that we're looking for here. For those who are less into the world of webdev,
being a W3C standard is about the closest thing you can get to 'official'. This
is hardly the only solution that exists right now, either, but it's by far the
most popular existing standard. Twitter themselves were even working on this
problem previously as part of their Bluesky project, which you might have heard
about in passing.

-----

If we solve this problem, we immediately solve the vast majority of the lock-in
problem that platforms currently offer. However, let's not get carried away with
ourselves and spread delusions of technoliberal utopia. Federation does *not*
solve all problems.

SMS can be thought of as a similar system. Your service provider gives you a
phone number, and the ability to use their services to talk to other people that
are connected to their network. In addition, they can talk to other service
providers who have their own networks, so you can send messages to other people's
phones even if they're on a different provider. This is a much more open and
permissive network, but it didn't stop Apple from holding the United States
hostage with iMessage.

iMessage chooses not to talk with other services because it's profitable not to.
Apple technically still supports SMS, so your iPhone can talk to anyone in the
world, but if you're not talking over iMessage, you're going to have a bad time.
In response to this, the rest of the industry have moved on from SMS to RCS,
which upgrades messaging standards worldwide to support properly rich media, but
Apple will still choose to ignore it despite Google's desperate marketing. Why?
Because, even in a world where everyone else provides equivalent service via RCS,
iMessage will *still* enjoy the benefit of not talking to RCS. Your iPhone could
even fully support RCS messaging, but as long as Apple keeps iMessage users in a
separate world from RCS users, it's all benefits for them. The only way to win
is to have the two worlds talk to each other properly.

This also brings up a related point about compatibility. iPhones technically
support a very small subset of RCS - specifically, they can understand SMS
messages, and pretty much nothing more. This makes for a terrible user
experience because Apple refuses to support more of RCS. This is analogous to
the problems we could experience if other social spaces *technically* support
some standards, but don't keep the support up-to-date or elect to only support
small parts of it as they see fit. Another example of this sort of shenanigan is
what happened with web browsers back in the Netscape/Internet Explorer wars.
Each browser supported the basic languages of the internet - HTML, CSS and
JavaScript - but each browser would offer their own exclusive features that you
couldn't get on the other, and try to get everyone using those exclusive
features in the hope they could extinguish their competition. As we all know now,
Microsoft won that war against Netscape, and it's so famous that the 'Embrace,
Extend, Extinguish' mantra is now known widely across the tech sphere. This must
never happen again.

The point here is that we need to make sure our social spaces work together at
more than a superficial level, and make sure that no single actor can control a
technically-free network. I'm not qualified to talk about what such solutions
look like, because this is creeping into political and legal territories, which
I'm not knowledgable about to give strong opinions on.

-----

There are two cyber-social freedoms: the freedom from platforms, and the freedom
from your friends' platforms.

We already have freedom from platforms, because the web is fundamentally free.

We should all work to have freedom from our friends' platforms too. Kill
lock-in. Connect the web of social spaces. Twitter and Cohost can stay, but they
better play nicely with their coevals.