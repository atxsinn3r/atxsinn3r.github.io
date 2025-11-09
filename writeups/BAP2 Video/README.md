# An Emotional Video I Made That Was Never Published

A long, long time ago... exactly 12 years ago from today (Nov 2025), I was at Rapid7 working on the Metasploit Project, and I was very into browser exploitation at the time. I was not good at it, but I was into it. This was an era when Microsoft Internet Explorer was a thing, and it was plagued by all these use-after-free bugs, which made headlines, and people made good money out of it.

The pentesters normally would deploy a browser exploit this way: first they would find out what browser people were using, hopefully getting the version too -- either this was learned before the engagement, or they would have to social engineer to find out. The intel was such a valuable thing to know, it was even a flag to collect at Defcon's SE contest.

After that, they would pick a good exploit against that environment. I'd say many browser exploits on Metasploit were made for IE between v7 - v11, Win XP - Win 7, so hopefully the target was on one of these. And then they would set up the exploit, and send it.

The deployment process was just a normal way of life for pen testers. But as a developer, I felt it was a lot of manual work, hideous, and somewhat error-prone.

Metasploit at that time already had this thing called Browser Autopwn, made by Egyp7. It was excellent at detecting the browser (including the version, and the plugins), but it was spamming all these exploits at the target and hope one of them would hit bullseye. Maybe it worked ok when it was published, but over time it would definitely make the browser quite unstable and crashy. It became clear it's better to just figure out what browser the target is on manually, and pick an exploit accordingly, like the pen testers had been doing.

So I decided to make Browser Autopwn 2, and I wanted to have a better deployment experience with these goals:

* The system learns the target (browser type, version, plugins)
* The system picks the most suitable exploit based on the browser information.
* I made multiple other features to make Browser Autopen 2 more useful, but these two above were the most important goals for me.

I also want to point out that maintaining a browser exploit kit is not easy, especially these are Nday exploits. The exploits wouldn't have a long shelf life, so you'd have to keep writing new ones to make the kit even useful. When Browser Autopwn 2 was developed, it had this in mind. So it was designed NOT to fire exploits recklessly if it decided there are no suitable exploits. See, I put a lot of little thoughts into it.

Not to sidetrack too much, but this was in 2013, and I was convinced this could be possibly done with AI. AI was not a thing back then. Nobody could prove (except maybe for DARPA) that AI could automate pen testing. Well it is 2025 now, if you're not investing in AI, you're not keeping up... maybe someone should pick up my idea again.

Anyway, back to what I was saying...

I did Browser Autopwn 2 in two stages:

* The first stage was creating a mixin called [BrowserExploitServer](https://github.com/rapid7/metasploit-framework/pull/2613). Basically this is an exploit mixin that allows the exploit writer to specify what conditions that work best for this exploit.
* The second stage was [Browser Autopwn2](https://github.com/rapid7/metasploit-framework/pull/5650). Basically it would arrange an exploit list to serve, and then depending what browser is visiting, it would redirect to the most suitable exploit. And the next one, and the next one, etc.

And it was out, and it achieved what I wanted it to achieve. It was cool. I also learned that even the NSA's TAO (Tailored Access Operations) unit was developing the same thing called FOXACID. From I can tell it was a bigger codebase. Of course, Metasploit's Browser Autopwn 2 was no match to that, but that leak confirmed I was seeing a need that the NSA was seeing, and I was pretty happy with that idea.

Like I said, I was pretty passionate with this thing. So I made a video about it, but the PR team rejected it. I remember it was because I had Jacob Appelbaum's footage in it and he was considered a controversial person, so my video was considered political. I didn't want to fight this too much but it was just misunderstood. I made this video with my heart and soul emphasizing that browser threats were emerging and it was serious. And I used a mix of footages to express what I wanted to say. They were my words.

That was about 12 years ago. All this should be pretty harmless by now. So here I am. I finally let go a piece of history, a museum piece, that I secretly kept with me for 12 years:

https://www.youtube.com/watch?v=oGaFUWiN3nU

Thank you all for the memory.



