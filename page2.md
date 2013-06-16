---
layout: default
title: Visual Spoofing
permalink: visual-spoofing/
---

# Unicode Security Guide
## _Visual Spoofing_ 

While Unicode has provided an incredible framework for storing, transmitting, and presenting information in many of our world’s native languages, it has also enabled attack vectors for phishing and word filters. Commonly referred to as ‘visual spoofing’ these attack vectors leverage characters from various languages that are visually identical to letters in another language.

To a human reader, some of the following letters are indistinguishable from one another while others closely resemble one another:

<span class="indent"> A&#x0391; &#x0410; &#x15C5; &#x15CB; &#x1D00; &#xFF21;</span>

To a computer system however, each of these letters has very different meaning. The underlying bits that represent each letter are different from one to the next.

## Table of Contents 
* [Prior Research](#prior) 
* [Attack Scenarios](#attack)
* [Defensive Options](#defense) 

## <a id="prior"></a>Prior Research
One of the most well-known attacks to exploit visual spoofing was the Paypal.com IDN spoof of 2005. Setup to demonstrate the power of these attack vectors, [Eric Johanson](http://www.shmoo.com/idn/) and The Schmoo Group successfully used a [www.paypal.com](http://www.paypal.com) lookalike domain name to fool visitors into providing personal information. The advisory references original research from 2002 by [Evgeniy Gabrilovich and Alex Gontmakher](http://www.cs.technion.ac.il/~gabr/papers/homograph.html) at the Israel Institute of Technology. Their original paper described an attack using Microsoft.com as an example.

Viktor Krammer, author of the [Quero Toolbar](http://www.quero.at/) for Internet Explorer, also presented additional research on these attack vectors and detection mechanisms in his [2006 presentation](http://www.quero.at/papers/idn_spoofing.pdf).  Additionally, the [Unicode Consortium](http://unicode.org) has been active at raising awareness of these issues in its security papers, and in providing recommended solutions.

## <a id="attack"></a>Attack Scenarios 

### <a id="domains"></a>Spoofing Domain Names
### <a id="vanity"></a>Fradulent Vanity URL's
### <a id="profanity"></a>Bypassing Profanity Filters
### <a id="ui"></a>Spoofing User Interface Dialogs
### <a id="ads"></a>Malvertisements
### <a id="email"></a>Forging Internationalized Email

## <a id="defense"></a>Defensive Options

## <a id="confusables"></a>The Confusables


### <a id="single"></a>Single-Script Confusables
### <a id="mixed"></a>Mixed-Script Confusables
### <a id="whole"></a>Whole-Script Confusables

## <a id="idna"></a>Internationalized Domain Names in Applications (IDNA)


### <a id="idna2003"></a>IDNA 2003
### <a id="idna2008"></a>IDNA 2008
