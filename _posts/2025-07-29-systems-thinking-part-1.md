---
title: "Book Review: Learning Systems Thinking, Part 1"
date: 2025-07-29
tags:
  - reading
categories:
  - Book Reviews
---

Last weekend I read part 1 of 4 of [_Learning Systems Thinking, Essential Nonlinear Skills and Practices for Software Professionals_](https://learningsystemsthinking.com/), by Diana Montalion. I picked up the book after seeing a Linkedin Post by [Vlad Mihalcea](https://vladmihalcea.com/) on this book. I've been interested in systems thinking ever since I saw a fascinating [talk](https://m.confoo.ca/en/2025/session/the-hidden-dance-of-complexity) by Michiel Rook at Confoo 2025.

Systems thinking is a new way of thinking about complex systems. It challenges the old "linear" way of thinking by replacing causation with emergent properties, analysis with synthesis, and overall encouraging thinking of a system as a whole. 

To me, a way of thinking that challenges the old centuries-old Cartesian ways sounded a lot like [postmodernism](https://en.wikipedia.org/wiki/Postmodernism), and the teachings of Derrida, Deleuze, Morin, and other philosophers, mainly French. The French have this thing, some might call it intellectualism, where if they write something that is complicated, they feel driven to make it seem as complicated as possible, so that readers will recognize the genius of the writer.[^1] And as such, much of this litterature is cryptic and freakishly hard to understand. Derrida wrote over 80 books, and some of them are more poetry than philosophy. Postmodern philosophy has its merits, but it's sometimes hidden under layers of academic varnish.

Because of my experience with postmodern philosophy, I was a bit wary when I started this book. But even when the author states certain paradoxes around studying systems thinking (the more you learn, the more you know that you don't know), it always stays grounded and within reach for mere mortals like myself. This made the book (at least Part 1) a rather accessible read.

One thing that stood out in this first part of the book is the number of citations from Donella Meadows, especially [_Thinking in Systems: A Primer_](https://en.wikipedia.org/wiki/Thinking_In_Systems:_A_Primer), and the [Donella Meadows Project](https://donellameadows.org/). These both sound interesting, and I might consider reading them if this book proves to be what it promises.

![book cover](/img/systems-thinking-cover.png)

## What is Systems Thinking (ST)?

Systems Thinking invites us to "think differently". It is a recognition that, in the "Systems Age", "as software becomes systems of software, we have reached the limits of our traditional ways of thinking." (p. 4) This different way is a departure from "linear thinking": the rational "if this, then that" mode of thinking that we all know and love. Linear thinking is necessary, but it can be a hinderance when reasoning about complex systems, such as the socio-technical systems we build.

The author gives this succinct definition of ST: "a system of foundational thinking practices that, when done together, improve nonlinear thinking skills." (p. 10) This is a fairly broad definition, but it gives a good idea that the point is to supplement "linear thinking."

## Two Examples of ST

The author gives two example of ST in action. They give a good idea of what ST is about.

First, she gives the example of a fictitious magazine (but many will recognize their struggles) that launches a website, but soon they realize they also have to support not only text, but also various types of content, and on multiple platforms (mobiles, desktops, mobile apps). As the platform grows, it becomes unbearably complex. And then the _coup de grâce_, the company between their core software goes out of business. This is the book's main case study, so she doesn't give the ST solutions right away. But she does hint at them, when she says tat the concepts used by the platform are ultimately what hinders it: "if [they] replace the software with a similar software, they will be making the problem worse, because now they will have spent millions of dollars on something that won't help them maintain competitive advantage." (p. 35)

Second, the author imagines a situation where two teams need their products to interact, but are finding it hard to cooperate. Where some would throw more managers at the problem, she takes another road, and talks about the company's hiring practices. She notes that when hiring, knowledge of algorithms and technical skills are rigorously tested through various challenges, but "soft skills", like cooperation and communication, are brushed aside. This creates teams that are technically masterful, but also creates an environment that is ripe for open conflict and poor collaboration.

## The Iceberg Model

To facilitate ST, she proposes what she calls the Iceberg Model, where the first layer of the iceberg is outside the water (picture the list below as the picture of an iceberg), and the remaining layers are hidden underwater:

1. Events
2. Patterns and Trends
3. Structure
4. Mental Models

She says that root causes of events are almost always mental models: "the things we believe are true that may (or may not) be true" (p. 44). At first, I was surprised that she mentioned root causes at all, because I thought of ST as a departure from things like root cause analysis. But if we take the example of an event where prod gets destroyed because a test change was pushed directly to `main`, it makes sense if what we think of as a root cause in ST is not just another event (like the `main` branch not being protected), but a mental model (like the notion that humans are always careful, or benevolent).

## General wisdom...

I'm still having trouble discerning the science of ST from just acquired wisdom, because there's a lot it to go around in this section of the book, starting with the emphasis on software systems as _socio_-technical systems. Gerald Weinberg said it best in the 1980s: "no matter how it looks at first, it's always a people problem" (in _The Secrets of Consulting_). Most staff+ software engineers know that software problems are often people problems. This is why ST encourages us to "view challenges as inherently sociotechnical."


From the two examples above, we can see that ST looks at software engineering issues and takes an orthogonal path to addressing them. We'll see if the rest of the book confirms my understanding, but so far I think a lot of it is just answering the question "in what world would this issue not exist?"

## ...that might not be shared by all

Montalion insists on the fact that ST solutions are often counterintuitive, and that they might not be shared by all (or by any). She says ST is hard, and is both an art and a science. As such, ST needs practice, and even experts end up taking wrong decisions based on ST.

Knowing this, one could think, "why would you even bother?" This reminds me of something Derek Muller, from YouTube channel [Veritasium](https://www.youtube.com/@veritasium), said about the scientific [Replication Crisis](https://en.wikipedia.org/wiki/Replication_crisis): "As flawed as our science may be, it is far and away more reliable than any other way of knowing that we have ([source](https://www.youtube.com/watch?v=42QuXLucH3Q))." Our understanding of reality is limited, and that's even more true of social phenomenon. So when a tool claims that it helps us understand it better, this "better" is not going to be perfect. I think that's what Kent Beck is getting at in his appreciation for the book:

> The book sells itself short. "Your life is not about to get easier" – yes, it is. You'll learn how ot stop making things worse by trying to make them better.

## Conclusion

If you've ever been on Youtube and admired the insult to human intelligence that are its thumbnails and its clickbaity titles, you've probably seen something like "STOP doing this WRONG", or "The biggest MISTAKE we all make". As much as I empathize with the content creators who are just trying to get their (sometimes really good) product out there, I hate this so much.[^2] But perhaps this book is the serious, scientific equivalent to the clickbait trope: "STOP thinking linearly about non-linear systems."

## Footnotes

<!-- prettier-ignore-start -->
[^1]: In fact, my experience is that it is often easier to read English translations of this literature, because Americans are allergic to bulls**t, so translators have to figure out what the author meant before they translate it. If some doubt my claim that a lot of this material is poetry, consider the grandiloquence of Derrida when he uses the word _viol_ (rape) to talk about how what someone says is never only the product of the speaker (in [_L'Écriture et la différence_](https://fr.wikipedia.org/wiki/L%27%C3%89criture_et_la_Diff%C3%A9rence).
<!-- prettier-ignore-end -->

