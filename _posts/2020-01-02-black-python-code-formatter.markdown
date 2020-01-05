---
layout: post
title:  "Black python code formatter"
date:   2020-01-02 13:37:00 +0200
categories: python
---

So, I was browsing the job offers of a company I'm particurly interested in working for and I noticed something odd. Never have a seen a job offer in which the company specifically talks about their code formatting, or while we're at it, their code in general. 

> To ensure a standardized code style we use the formatter black.

Okay. So [black](https://github.com/psf/black), hu?
Can't say I don't like the colour, but I never heard of the code formatter, so I went ahead and watched Łukasz Langa's talk [Life Is Better Painted Black, or: How to Stop Worrying and Embrace Auto-Formatting](https://www.youtube.com/watch?v=esZLCuWs_2Y). To say that it was a revelation is an understatement.  

When I was starting out with python I didn't really know about [PEP-8](https://www.python.org/dev/peps/pep-0008/), and to be honest I didn't really care. I was hyped that what I was writing code that worked and that's it. Once exposed to other peoples code via github or the likes I noticed that they wrote code differently yet similar, and that I could quickly understand what their code was doing. So I read [PEP-8](https://www.python.org/dev/peps/pep-0008/) and tried to stick to it as much as I could without looking it up every so often.  

I eventually started working my first job where I was able to use Python professionally. We were a small team of five each having their own tasks and responsibilities, but working on the same product eventually. We had work agreements on a lot of stuff, from testing to code reviews, yet code formatting was that one agreement, where there was no agreement. While everybody claimed to know [PEP-8](https://www.python.org/dev/peps/pep-0008/) and claimed to stick to it, nobody actually did. When you point it out, people get defensive, there are arguments over line-length and line-breaks - all which is opinionated and finally, just wasted time.  

> Black is the uncompromising Python code formatter. By using it, you agree to cede control over minutiae of hand-formatting. In return, Black gives you speed, determinism, and freedom from pycodestyle nagging about formatting. You will save time and mental energy for more important matters.

Black takes away the discussion. Does it stick to PEP8? Na, not completely. Is it opinionated? Hell yes! Does it matter that it is? No, not at all. As [Dusty Phillips](https://smile.amazon.com/s/ref=nb_sb_noss?url=search-alias%3Daps&field-keywords=dusty+phillips) put it:

> *Black* is opinionated so you don't have to be.

You can try it out yourself: [black.now.sh](https://black.now.sh/).