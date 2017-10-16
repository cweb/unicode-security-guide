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

Viktor Krammer, author of the [Quero Toolbar](http://www.quero.at/) for Internet Explorer, also presented additional research on these attack vectors and detection mechanisms in his [2006 presentation](http://www.quero.at/papers/idn_spoofing.pdf).  Additionally, the [Unicode Consortium](https://unicode.org) has been active at raising awareness of these issues in its security papers, and in providing recommended solutions.

## <a id="vectors"></a>Summary of Vectors
The phenomena of 'visual spoofing' may be malicious and deliberate or benign and accidental.  There have been cases where a choice of font displayed a sequence of characters in an unintended way, just as there have been cases where Unicode characters did not display properly.  The following list attempts to capture the major vectors:

__Non-Unicode lookalikes__

Simple characters or character combinations can look like something else.  For example, the letters "r" and "n" together can look like the letter "m".  E.g. <span class="spoof">"rn"</span>.  Also, the number <span class="spoof">"0"</span> can look like the letter <span class="spoof">"O"</span>, the number <span class="spoof">"1"</span> can look like the letter <span class="spoof">"l"</span>, and so on.

__Unicode Confusables__

The Unicode Confusables are discussed in detail later in this document.  In short, these are the diverse array of non-ASCII Unicode characters which are easily confused with characters across languages.

__ The Invisibles__

Discussed later in this document, these are characters which have no visual appearance and minimal spacing if any spacing at all.  Hence, they are visually non-existant.  

__Problematic font-rendering__

Fonts are ultimately responsible for the visual display of characters, and can sometimes render glyphs confusingly, or as empty white space.  There are numerous examples of this, just one of which is described below:

<table>
<thead>
 <tr>
   <td>Character sequence</td>
   <td>Should appear as</td>
   <td>Might appear as</td>
 </tr>
</thead>
<tbody>
 <tr>
   <td>U+00B7 U+0041 U+0338</td>
   <td><span class="uchar">A&#x0338;</span></td> 
   <td><span class="uchar">A/</span></td>
 </tr>
</tbody>
</table>

__Manipulating combining-marks__

Combining marks can be stacked or re-ordered in a myriad of ways.  Consider the following table which illustrates just one way that combining marks can be stacked (using one directly after another). The table also shows an example of how combining marks can be re-ordered in a different sequence, but still have the same visual appearance.

<table>
<thead>
 <tr>
   <td>Character sequence</td>
   <td>Appears as</td>
 </tr>
</thead>
<tbody>
 <tr>
   <td>U+006F U+0304</td>
   <td><span class="uchar">o&#x0304;</span></td> 
 </tr>
 <tr>
   <td>U+006F U+0304 U+0304</td>
   <td><span class="uchar">o&#x0304;&#x0304;</span></td> 
 </tr>
 <tr>
   <td>U+006F U+0336 U+0335</td>
   <td><span class="uchar">o&#x0336;&#x0335;</span></td> 
 </tr>
 <tr>
   <td>U+006F U+0335 U+0336</td>
   <td><span class="uchar">o&#x0335;&#x0336;</span></td> 
 </tr>
</tbody>
</table>

__Bidi and syntax spoofing__
Another interesting vector uses the 'bidirectional' properties of certain characters, also known as 'bidi'.  

## <a id="attack"></a>Attack Scenarios 

A variety of scenarios exist where visual spoofing may be used to attack and exploit people.  This section looks at a few.

### <a id="domains"></a>Spoofing Domain Names
Domain names represent an interesting attack vector because their mere visual appearance inspires trust in a brand.  The following image represents what visually appear as two identical domain names, however, the second contains the <span class="uchar">U+0261 LATIN SMALL LETTER SCRIPT G</span>.

<img class="center" src="{{ site.url }}/img/spoof-google.png" />

The tricky part about presenting domain names is that they often get simply a glance, if any look at all.  The following image represents two domain names which might be visually similar 'enough' to fool someone, yet not identical.

<img class="center" src="{{ site.url }}/img/spoof-mozilla.png" />

Finally, characters that appear to be syntactic elements, such as <span class="uchar">U+FF89 HALFWIDTH KATAKANA LETTER NO</span> which resembles the forward-slash path-separator, can be troublesome.  In the following image, this character is used in the subdomain label of a domain name, but appears to be a path-separator.

<img class="center" src="{{ site.url }}/img/spoof-slash.png" />

### <a id="vanity"></a>Fradulent Vanity URL's

A social networking service wants to allow vanity URL’s to be registered using international characters such as <span class="uchar">www.foo.bar/&#x0444;&#x0443;</span> but perceives too great a risk from the variety of ways that the URL could be subject to visual fraud and confusion. Because Unicode characters are well-supported in the path portion of a browser’s URL display, a well-crafted vanity URL could easily fool victims and be the landing page for a phishing attack.  In fact, it's often unnecessary to use Unicode - in some cases, the number one "1" can appear as the letter "l", and in certain fonts the sequence "rn" can appear as the letter "m".

### <a id="profanity"></a>Bypassing Profanity Filters

An email or forum system needs to prevent violent and profane words from being used. It's well-known that there are trivial ways to bypass such filters, including using spacing and punctuation between letters in a word (e.g. c_r_a_p), or slight misspellings which give the same effect (e.g. crrap), to name just a couple.  There’s also the possibility of using confusable characters which have no visual side-affect (e.g. crap) written entirely in another script (or a mix of scripts).  

### <a id="ui"></a>Spoofing User Interface Dialogs

Security decisions are often presented to end users in the form of dialog boxes consisting in part of user-supplied input. For example: 

* When a user downloads a file through a Web browser, they’re asked to confirm their decision, often with the filename as a part of the dialog's content. 
* When a user tries to launch an untrusted application they may also be presented with a dialog box asking for confirmation. 
* A social networking site may ask its users for confirmation before redirecting them to an off-site URL, often with the URL making up the dialog's content. 

In any of these cases, a clever attack may use special BIDI or other characters that reverse the direction of text, or otherwise manipulate the text in a way that may confuse or fool the end users.

Consider the following image which shows the Windows Explorer program.  What appears to be a plain text file ending in the ".txt" file extension, is actually an executable file ending in the ".exe" extension.  Fortunately, Windows Explorer recognizes the true file type, which it has listed as "Application".

<img class="center" src="{{ site.url }}/img/spoof-win-explorer-file.png" />

In another example, the <span class="uchar">U+FEFF ZERO WIDTH NO-BREAK SPACE</span> character, also known as the Byte-Order Mark, or BOM, acts as an 'invisible' character.

<img class="center" src="{{ site.url }}/img/spoof-win-explorer-folder.png" />

Invisible characters present their own interesting dymanics and applications.  As seen in the image above, Windows Explorer presents what appears to be two folders with identical names, whereas a default command prompt does not properly display the BOM, and so presents it as an empty box.

### <a id="ads"></a>Malvertisements

Advertising network's often need to protect brand name trademarks from being registered or used by anyone other than their owner. This threat might be mitigated through filters, human editorial inspection, or a combination of the two.  An attacker could place an malicious phishing ad that bypasses trademark filters by using confusable characters. For example “Download Microsoft Windows 8 Service Pack 1 here” where the trademarked name 'Microsoft Windows' was crafted using non-English script, or even using invisibile characters.

### <a id="email"></a>Forging Internationalized Email

Email addresses and the SMTP protocol has long been confined to ASCII, however, standards work through the <a href="http://www.ietf.org">IETF</a> was concluded in 2013 by the <a href="http://datatracker.ietf.org/wg/eai/charter/">Email Address Internationalization Working Group</a>.  The EAI effort delivered documentation for integrating UTF-8 into the core email protocols, as well as advice to EAI deployment in client and server software.  In preparing for the transition, email client engineers and designers will need to anticipate and handle the case of visually identical email addresses, among other issues.  If left unhandled, then end users could easily be fooled. Digital certificates would provide a good mechanism for proving authenticity of a message; however such certificates also support Unicode and are vulnerable to the exact same attacks.

## <a id="defense"></a>Defensive Options
All does not seem lost.  While

## <a id="confusables"></a>The Confusables
Throughout Unicode, the characters that visually resemble one another are referred to as <strong>the confusables</strong>.  The Unicode Consortium has documented this phenomena in both <a href="https://www.unicode.org/reports/tr36/">Technical Report 36</a> and <a href="https://www.unicode.org/reports/tr39/">TR 39</a>.  

It is TR 39 specifically which provides links to the data files comprising the confusables, such as <a href="https://www.unicode.org/Public/security/revision-05/confusables.txt">confusables.txt</a> which provides a mapping for visual confusables.

The Unicode Consortium has also provided <a href="https://unicode.org/cldr/utility/confusables.jsp">Unicode Utilities: Confusables</a> which takes an input string and produces visually confusable strings generated using the prior mentioned data files.

### <a id="single"></a>Single-Script Confusables


### <a id="mixed"></a>Mixed-Script Confusables
### <a id="whole"></a>Whole-Script Confusables

### <a id="whole"></a>The Invisibles

<img class="center" src="{{ site.url }}/img/uchar-180E.png" />
<img class="center" src="{{ site.url }}/img/uchar-feff.png" />

## <a id="idna"></a>Internationalized Domain Names in Applications (IDNA)


### <a id="idna2003"></a>IDNA 2003
### <a id="idna2008"></a>IDNA 2008
