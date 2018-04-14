---
layout: post
title: PyconFR 2016 - I was there
description: A short feedback about the PyconFR 2016 at Rennes
date: 2016-10-22
tags:
    - python
    - conference
---

Last week-end, I went to the PyconFR 2016 at Rennes. It was the first Python event I went to since I have started to work with this language.

I must say, I was quite eager to see and meet people from the Python community. To be totally honest, when I started to work with Python, I had a mixed feeling about the Python community. I love the fact that it is almost only people working on it in their free time without the drive of big companies behind (I am not saying that companies are not involved here, only that most contributors do not have any outside influences from companies). However it seems sometimes that it is quite a mess. For example, there is not one accepted solution for packaging or even for dependencies management (easy_install, pip, setuptools, distutils, distutils2, distribute). The bases of the language are easy to get but the tooling suite is a mess.

With this preamble, let's start talking about the conferences.

I did not go to the code sprint the first 2 days because of work but I hope to be able to attend one in the future. It seems to be a great place to meet and learn from other coders.

My first point is about the dates. It is refreshing to have a conference on a week-end. I am working in a startup where something new happens everyday for the moment and it is difficult to take 2 consecutive days off. So having the conferences on the week-end was perfect for me (sorry for you if you had a family life ;).

My second point is about the organization. There I want to say a big **BIG UP** for the [AFPY](https://www.afpy.org/) and [Telecom Bretagne](http://www.telecom-bretagne.eu/). It was a perfect run. The conferences started and ended on time. You had a small buffet with coffee or tea all day long, a breakfast in the morning and food trucks for lunch. There was a room locked by the AFPY were you could leave your luggages. All in all, It was great despite the rain on the first day.

Now I will describe each conferences I attended during these 2 days.

I must say that I did not like the format of the conferences at all. You had 2 durations : 25 minutes and 45 minutes. Most of the conferences lasted for 25 minutes included 5 minutes of question. So 20 minutes to talk about a subject is much too short. It is the worst thing I have to report about this event. Most of the talks ended as a small introduction to a subject without any depth. This was quite frustrating if you already have some background and experience in computer science. The standard duration should have been 45 minutes for all talks.

I will now give my opinion on the conferences I attended.

You had 3 tracks of conferences. If I were to describe them, I would say :

1. Around Python : libraries, tools, ...
2. Machine learning with Python
3. Python internals

I went into tracks 1 and 3 mostly. It has been 1 year since I have started to work with Python. So I don't want to tackle too many things and I stayed focused on Python and its environment without going into the machine learning side. Sorry for the French titles, most of the talks were in French (and by the way, sorry for my spelling and grammar mistakes in English ;).

* [LA PROGRAMMATION ASYNCHRONE AVEC PYTHON.](https://2016.pycon.fr/pages/programme.html#La%20programmation%20asynchrone%20avec%20Python.) : It mostly talked about AsyncIO and only the basics. I am disappointed that it did not present alternative solutions like gevent. Moreover the speaker did not talk about the limits of the coroutine (you can't do computation intensive tasks inside as it blocks other coroutines) and how to work around them.
* [POURQUOI, MAIS POURQUOI, ASYNC ET AWAIT ONT ÉTÉ INCLU DANS PYTHON 3.5 ?](https://2016.pycon.fr/pages/programme.html#Pourquoi,%20mais%20pourquoi,%20async%20et%20await%20ont%20été%20inclu%20dans%20Python%203.5%20?) : When you read the title, you are waiting for a pure python conference. However it ended as another conference explaining how AsyncIO is great without explaining when it is useful.
* [VOYAGE AU CENTRE DU MONDE CPYTHON](https://2016.pycon.fr/pages/programme.html#Voyage%20au%20centre%20du%20monde%20CPython) : a great talk on advised steps you can take to start contributing in Python core. Very interesting and informative. I recommend this one (a 45 minutes talk. What a surprise ;).
* [IMPORT ET COMPAGNIE](https://2016.pycon.fr/pages/programme.html#Import%20et%20Compagnie) : another great talk about what is happening when you write `import something`. It goes very deep in how the interpreter find the package or module you are importing.
* [PYPY: PYTHON FASTER THAN PYTHON](https://2016.pycon.fr/pages/programme.html#PyPy:%20Python%20faster%20than%20Python) : Interesting talk about another Python implementation. Educative. But let's be honest, who uses other Python flavors than CPython ?
* [À LA DÉCOUVERTE DU BYTECODE CPYTHON !](https://2016.pycon.fr/pages/programme.html#À%20la%20découverte%20du%20bytecode%20CPython%20!) : In 20 minutes, the speaker succeeded in giving a lot of information on how the bytecode Python works. A lot of information and a good starting point if you are interested in the internal of the interpreter.
* [LES DESSOUS DU PORTAGE D'ANSIBLE À PYTHON 3](https://2016.pycon.fr/pages/programme.html#Les%20dessous%20du%20portage%20d'Ansible%20à%20Python%203) : One of the worst conference. I don't know if it was Redhat/Ansible or the speaker who did not want to be filmed but this was a good choice as it will prevent others to waste their time by watching an empty conference (I am vehement here because I had high hope for this talk and was very disappointed). If you have already migrated something from Python2 to 3, you know that there are issues with encoding or some function names (and other things ;). This last sentence summarizes this talk. Why did you not show some migrated code confronted with its old version ? How did you organize this migration ? How many people were working on it ? How did you manage to continue evolving the code base during the migration ? These are the kind of questions I would love to have an answer to but all I had was : we had some issues with encoding ...
* [LIRE & ÉCRIRE LA DOC](https://2016.pycon.fr/pages/programme.html#Lire%20&%20Écrire%20la%20Doc) : A philosophical approach to documentation writing. Not much about the tools but a great deal of information about how to organize a documentation and drive a team to maintain it. All in all, an enjoyable talk.
* [L'ENFER DU PACKAGING PYTHON](https://2016.pycon.fr/pages/programme.html#L'Enfer%20du%20packaging%20Python) : Another disappointing talk, when you read the title, you are waiting for THE PACKAGING SOLUTION. All you get are some tips with pip and setuptools that most Python developers know about.
* [PACKAGING PYTHON WHEEL ET DEVPI](https://2016.pycon.fr/pages/programme.html#Packaging%20Python%20Wheel%20et%20Devpi) : This one was disappointing too. I was focused on the WHEEL part in the title and I was waiting for a talk about the wheel format. Finally, it was only a presentation of the devpi solution.
* [WAREHOUSE - THE FUTURE OF PYPI](https://2016.pycon.fr/pages/programme.html#Warehouse%20-%20the%20future%20of%20PyPI) : Nothing technical here. But it is a refreshing talk about the transformation of the PyPI web interface with issues on ergonomics, design, ....
* [DES NOUVELLES DU FRONT !](https://2016.pycon.fr/pages/programme.html#Des%20nouvelles%20du%20Front%20!) : Write Python and transpile it to javascript in this talk. Interesting to know that it exists. But : Why, why are people wasting their time in transpiling one language to another. It was proved many time that in the end native applications are what customers want to use.
* [WEBPUSH NOTIFICATIONS WHAT? WHY? HOW?](https://2016.pycon.fr/pages/programme.html#WebPush%20notifications%20What?%20Why?%20How?) : Another introduction talk about web push notification. Just read a tutorial on the web and it will be the same. It could have been a great talk with examples, push server benchmarks or recommendations, architecture to scale, ...
* [FAUT-IL ÊTRE MASOCHISTE POUR UTILISER IPV6 ?](https://2016.pycon.fr/pages/programme.html#Faut-il%20être%20masochiste%20pour%20utiliser%20IPv6%20?) : One if not the best conference I attended. A good balance between code, explanation, difficulties and solutions when working with IPV6 in Python. I will subscribe to the IPV6 MOOC the speakers introduced.
* [OUTILS D'ANALYSE STATIQUE](https://2016.pycon.fr/pages/programme.html#Outils%20d'analyse%20statique) : Another beginner talk about code quality tools. Just read a tutorial on the web.
* [PYTHON ET LA SÉCURITÉ : DE L'INTERPRÉTEUR AU DÉPLOIEMENT](https://2016.pycon.fr/pages/programme.html#Python%20et%20la%20sécurité%20:%20de%20l'interpréteur%20au%20déploiement) : I have 2 contradictory feelings about this talk. There were a great deal of information and things to learn but I still have the feeling in the end that it was only obvious things you have in all languages and in all professional applications. I think it was a 1 hour talk intended. Maybe it would have been better in this format.
* [INFRASTUCTURE MODERNE POUR LE DÉVELOPPEMENT EN ÉQUIPES](https://2016.pycon.fr/pages/programme.html#Infrastucture%20moderne%20pour%20le%20développement%20en%20équipes) : Finally a talk about a real world solution. Very interesting (with a great speaker I work with sometimes). It explained how a company implemented continuous integration and staging delivery. It listed some softwares I did not know and bot configurations.
* [AU SECOURS, ON N'A PAS DE PROJET PYTHON DANS MA BOÎTE](https://2016.pycon.fr/pages/programme.html#Au%20secours,%20on%20n'a%20pas%20de%20projet%20Python%20dans%20ma%20boîte) : How to use Python in your company when it is not the primary language ... Okay you can do some scripts ... But why did you not use the main language (or secondary there is always one) of your company ?
* [TEST TOUT TERRAIN](https://2016.pycon.fr/pages/programme.html#Test%20Tout%20Terrain) : ... What to say ... After one week, I don't remember what it was about ... Most probably I did not learn anything ...
* [PYTHON, UN LANGAGE À LA NOIX POUR LA PROGRAMATION FONCTIONELLE ? ESSAYEZ COCONUT !](https://2016.pycon.fr/pages/programme.html#Python,%20un%20langage%20à%20la%20noix%20pour%20la%20programation%20fonctionelle%20?%20Essayez%20coconut%20!) : Coconut a functional python paradigm. I am not convinced by the word functional in the title. It seems to me that some people wanted some new keywords in their Python code ...
* [HYPOTHESIS: TESTEZ MOINS MAIS TESTER MIEUX EN VOUS CONCENTRANT SUR LES PROPRIÉTÉS](https://2016.pycon.fr/pages/programme.html#Hypothesis:%20testez%20moins%20mais%20tester%20mieux%20en%20vous%20concentrant%20sur%20les%20propriétés) : A promising unit tests solution to generate data to test edge cases automatically. I did not know this library. A good conference to end these 2 days.

Ouf ... It was quite a sprint ... As I said, I would have preferred to listen to half of these conferences if it were to go more in depth on some subjects. I leave this PyconFR with the impression I did not learn much. Some of the speakers came from computer service companies : Where were the talks about architectures they implemented for their customers ? And how Python help them meet their need ? If the format of the conferences are the same next year, I won't attend, maybe I will focus on the code sprint instead and watch some videos on Youtube later.

This conclude my feedbacks on PyconFR 2016. One more time, **BIG UP** AFPY and Telecom Bretagne, thank you for these 2 days.

![](/assets/img/pycon/people.jpg)

_You can play find Jonathan (I had a hard time finding myself even knowing where I was at that time, I am hidden behind an arm and only half of my face is visible_
