---
layout:	post
title:	"Building projects with MS Build"
date:	2015-11-28 14:55:25 +0200
categories:	Development
tags:	[development, build, ms-build]
---

***Abstract:** Using MS Build to compile your entire solution. *

* TOC
{:toc}

#  Introduction (The back story)

If you don't like waffle, you can skip this part :P

A few years back I went to Microsoft's Tech-Ed in Durban, South Africa, and attended a session with Brian Keller on MS Test Manager and acquired a book on MS Build. I was still relatively junior at the time, but applied MS Build to my work and have gleaned a repertoire for it.

MS Build is a build scripting tool for .NET similar to the likes of Ant and Make.

The "traditional" way of building .NET projects was that you download a zip file with the projects and solution file, open it up is Visual Studio and ask it to build. Coming from the Linux world where Make is pretty standard, this is not a great way of compiling projects, often because Visual Studio formats would change between versions. This then meant you could be greeted by a project upgrade screen which may fail, and really, who wants to wait for Visual Studio to load anyway? Half the time I scope out the code with Notepad++.

While this might be okay (albeit not ideal) for little demo projects and code snippets, its a big problem for larger systems - especially when they are too big to "fit" in a single Visual Studio Solution, with other dependencies that need to be compiled and and and...

#  Enterprise Level Builds

So, the problem with enterprise software is that often you have too many projects to fit into a single solution without crashing Visual Studio or grinding your machine to halt. But at the same time, you have modules which are dependant on other ones which is normally managed by Visual Studio.

My standard goto for this problem is a structured scheme for MS Build that allows you to easily compile from command line, resolving the required dependencies and doing whatever other pre and post build steps you might need.



