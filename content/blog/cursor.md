---
title: Cursor AI
description: A look at cursor AI
date: 2025-02-09
tags:
  - AI
  - Developer productivity
---

[Cursor AI](https://www.cursor.com/) is an IDE, a fork of visual studio, that includes powerful AI features. With Cursor AI you can ask the AI to generate code changes directly in the source files and this is the key feature that has been missing with all previous AI copilots.

I gave it a try.

I decided to build a React website from scratch, including user login, editing of profile information, search capabilities etc. with the help of cursor AI. I have limited experience with React, so being slightly out of my comfort zone made it a perfect fit to try out the AI capabilities. Also, since building a React website involves building many standard components that has been widely documented and discussed on the internet for many years this is something an AI is expected to be very good at.

## Verdict

- It is really fast to get going and have something up and running fast, however things quickly get more difficult as the code base evolves, handling the context becomes harder and keeping a consistent code style is challenging. E.g. composer seem to know about a lot of the code base even if files are not added explicitly to the context, but what are the exact rules?
- Most of the time understands the questions and generates code that does the right thing.
- Really good at generating mock data.
- Besides making changes directly in the code it can also make instructions about terminal commands that must be executed, e.g. to install needed dependencies, and the commands can be run by the click of a button!
- Auto-complete is really spot on.
- Insufficient understanding of how different versions of libraries relate to each other and handling breaking changes. What code works with which versions of a library, it sometimes generates code that doesn't work with the version of dependencies I had installed.
- Is not up to date with the latest versions of all libraries, it made false claims about "latest version of this library is x.x.x".

## From zero to a hundred

From a clean slate I asked it to generate new React app for me to get going. It generated everything for me and gave me instructions on how to spin it up locally and everything worked! I later realised that the create-react-app command that it had used to generate the app was actually deprecated and no longer recommended by the authors.

Going from there the workflow was to ask the composer to make an addition, inspect the proposed changes, correct any misunderstandings about what I had wanted it to do, that happened very rarely though, apply the changes to the code base and then make any corrections that I saw fit and finally making a git commit. Rinse and repeat.

Before Cursor AI each iteration in the workflow would have consisted of an investigation phase prior to the implementation phase, but with cursor AI I would ask it to suggest the changes and I would then only investigate if the ambiguity of the solution was high. This made me much more efficient and able to stay in the flow. Seeing code generated in small iterations is also a very efficient way to learn.

## Caveats

*Some of the following caveats could probably be circumvented by doing better prompting, e.g. with a more customised pre instruction and doing better management of the context in the chat and composer, i.e. ensuring that all relevant files have been shared at before asking a question.*

### No free lunch

Sometimes it makes mistakes and you'll need to figure out why something isn't working.
It can often help you out if you put in the error message in the chat, but sometimes you need to debug and understand the problem yourself.
This is what you normally do anyway, but due to the limitations of the AI it can sometimes get you into some weird situations that you would end up in if you had done it yourself. In other words, you'll need to understand what's going on in the code and have or gain a basic understanding of the language, frameworks/libraries you're working with. This is really not a problem and the AI can help you and guide you along the way to gain that understanding, but just to say that this is not a magic bullet that will make any junior developer be able produce quality code from day 1.

### The AI is not a senior at your company

The AI will generate code for you, but it doesn't have the big overview or vision of where you're going, it might be a senior engineer, but it doesn't know everything about you, your company and it hasn't seen all of your code and participated in the discussions about idioms and best practices that you and your fellow engineers have had. Supplying all of that to the AI so that it can guide you in all decision making might not be worth the effort?

The AI was not aware of the larger design and style of the existing code base, e.g. in which files would it be appropriate to put the generated code. It would even be inconsistent in the code it is generating itself. Initially it would put all `fetch` calls into a separate js file to separate that code from the react components/pages but later it would start putting fetch calls directly in there.

It also doesn't necessarily consider all possible ways of solving a task so it might not solve a problem in the most effective or appropriate way
in the given situation. E.g. if you want to do proximity search and are just toying with a tiny project perhaps it is not necessary to use a db that supports proximity search. You'll need to know which design decisions are the right for you in the given situation and then ask the AI to help you with the boring work.

### Context management is an unresolved issue

Like with other AI copilots dealing with the context is still a pita. It forgot that I was using sqlite and generated code that works for mysql. At some point it generated code in a new file which is already there in an existing file (setting up the database). It was of course my own mistake as I had not shared all the relevant files in the context, but I was in general struggling a bit remembering to keep the context up to date at all time, and it caused some confusion.

Although it was fairly manageable on this small toy project dealing with the limitation of the context window on real world projects, which has 100+ files, is a major issue. Until the AI copilots become better at managing the context, e.g. automatically discovering relevant files and adding them to the conversation, the applicability of the AI copilots will be very limited in a professional setting.

## Man vs machine

The AI tools keep getting better and better, so what's left for us to do? Are we just going to be gathering and understanding requirements and feeding those into an AI agent that will take care of everything from there?

I don't think so.

At the end of the day when shit hits the fan an engineer must be able to debug the system, and not being familiar with the code base, even with all the AI help you can think of, would make debugging it extremely difficult. Therefore, I think we'll continue to see AI as just another tool in the toolbox, irrespective of how good it is going to get.

So what's the learning experience going to be like for a junior in the future when you can simply generate working code?
As an engineer you'll be expected to deliver fast and reading though code that is working or reading up on related documentation is an investment in personal growth but not necessarily productive. So what would incentivise an engineer to do this?

The reason the tool is useful to me as an engineer is because I'm able to be critical of the generated code and make adjustments that I see fit, but the reason I can do that is because I've been coding, without the help of an omnipresent AI copilot, for decades.

Perhaps in the future software engineers will mostly do research projects, where a new tool or way of doing things is investigated and POC'ed and if deemed successful the learnings will get fed into the AI ecosystem and enter the development space that way?

Hopefully we will find a way for man and machine to live in harmony.
