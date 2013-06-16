---
layout: default
title: Background
permalink: background/
---

# Unicode Security Guide
## _Background_ 

The Unicode Standard provides a unique number for every character, enabling disparate computing systems to exchange and interpret text in the same way.

## Table of Contents 
* [History](#history) 
* [Introduction](#intro)
    * [Code Points](#cp)
* [Character Encoding](#encoding) 
* [Character Escape Sequences and Entity References](#escape)
* [Security Testing Focus Areas](#testing)

## <a id="history"></a>Brief History of Character Encodings
Early in computing history, it became widely clear that a standardized way to represent characters would provide many benefits. Around 1963, IBM standardized EBCDIC in its mainframes, about the same time that ASCII was standardized as a 7-bit character set.  EBCDIC used an 8-bit encoding, and the unused eighth-bit in ASCII allowed OEM's to apply the extra bit for their proprietary purposes. 

This allowed OEM's to ship computers and later PC's with customized character encodings specific to language or region. So computers could ship to Israel with a tweaked ASCII encoding set that supported Hebrew for example. The divergence in these customized character sets grew into a problem over time, as data interchange become error-prone if not impossible when computers didn't share the same character set. 

In response to this growth, the International Organization for Standardization (ISO) began developing the ISO-8859 set of character encoding standards in the early 1980's. The ISO-8859 standards were aimed at providing a reliable system for data-interchange across computing systems. They provided support for many popular languages, but weren't designed for high-quality typography which needed symbols such as ligatures. 

In the late 1980's Unicode was being designed, around the same time ISO recognized the need for a single character encoding framework, what would later come to be called the Universal Character Set (UCS), or ISO 10646. Version 1.0 of the Unicode standard was released in 1991 at almost the same time as UCS was made public. Since that time, Unicode has become the de facto character encoding model, and has worked closely with ISO and UCS to ensure compatibility and similar goals.

## <a id="intro"></a>Brief Introduction to Unicode
The Unicode framework can presumably represent all of the worlds languages and scripts, past, present, and future. That's because the current version 5.1 of the Unicode Standard has space for over 1 million code points. A code point is a unique value within the Unicode code-space. A single code point can represent a letter, a special control character (e.g. carriage return), a symbol, or even some other abstract thing.

### <a id="cp"></a>Code Points 
A code point is a 21-bit scalar value in the current version of Unicode, and is represented using the following type of reference where NNNN would be a hex value: 

<span class="indent">U+NNNN</span>

The valid range for Unicode code points is currently U+0000 to U+10FFFF.  This range can be expanded in the future if the Unicode Standard changes. The following image illustrates some of the properties or metadata that accompany a given code point.

Code point U+0041 represents the Latin Capital Letter A. It's no coincidence that this maps directly to ASCII's value 0x41, as the Unicode Standard has always preserved the lower ASCII range to ensure widespread compatibility. Some interesting things to note here are the properties associated with this code point:

* Several categories are assigned including a general 'category' and a 'script' family.
* A 'lower case' mapping is defined.
* An 'upper case' mapping is defined.
* A 'normalization' mapping is defined.
* Binary properties are assigned.

This short list only represents some of the metadata attached to a code point, there can be much more information. In looking for security issues however, this short list provides a good starting point.

## <a id="encoding"></a>Character Encoding
A discussion of characters and strings can quickly dissolve into a soup of terminology, where many terms get mixed up and used inaccurately.  This document will aim to avoid using all of the terminology, and may use some terms inaccurately according to the Unicode Consortium, with the goal of simplicity. 

To put it simply, an encoding is the binary representation of some character.  It’s ‘bits on the wire’ or ‘data at rest’ in some encoding scheme.  The Unicode Consortium has defined four character encoding forms, the Unicode Transformation Formats (UTF):

1. UTF-7
   Defined by RFC 2152.
1. UTF-8
   Each Unicode code point is assigned to an unsigned byte sequence of one to four bytes in length.
1. UTF-16
   Each Unicode code point is assigned to an unsigned sequence of 16 bits.  There are exceptions to this rule, see the discussion of surrogate pairs below.
1. UTF-32
   Each Unicode code point is assigned to an unsigned sequence of 32 bits with the same numeric value as the code point.

Of these four, UTF-7 has been deprecated, UTF-8 is the most commonly used on the Web, and both UTF-16 and UTF-32 can be serialized in little or big endian format.

A character encoding as defined here means the actual bytes used to represent the data, or code point.  So, a given code point <span class="uchar">U+0041 LATIN CAPITAL LETTER A</span> can be encoded using the following bytes in each UTF form:

<table>
 <thead><tr>
  <td>UTF Format</td>
  <td>Byte sequence</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>UTF-8</td>
  <td>&lt; 41 &gt;</td>
 </tr>
 <tr>
  <td>UTF-16 Little Endian</td>
  <td>&lt; 41 00 &gt;</td>
 </tr>
 <tr>
  <td>UTF-16 Big Endian</td>
  <td>&lt; 00 41 &gt;</td>
 </tr>
 <tr>
  <td>UTF-32 Little Endian</td>
  <td>&lt; 41 00 00 00 &gt;</td>
 </tr>
 <tr>
  <td>UTF-32 Big Endian</td>
  <td>&lt; 00 00 00 41 &gt;</td>
 </tr>
</tbody></table>

The lower ASCII character set is preserved by UTF-8 up through U+007F.  The following table gives another example, using <span class="uchar">U+FEFF ZERO WIDTH NO-BREAK SPACE</span>, also known as the Unicode Byte Order Mark.

<table>
 <thead><tr>
  <td>UTF Format</td>
  <td>Byte sequence</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>UTF-8</td>
  <td>&lt; EF BB BF &gt;</td>
 </tr>
 <tr>
  <td>UTF-16 Little Endian</td>
  <td>&lt; FF FE &gt;</td>
 </tr>
 <tr>
  <td>UTF-16 Big Endian</td>
  <td>&lt; FE FF &gt;</td>
 </tr>
 <tr>
  <td>UTF-32 Little Endian</td>
  <td>&lt; FF FE 00 00 &gt;</td>
 </tr>
 <tr>
  <td>UTF-32 Big Endian</td>
  <td> &lt; 00 00 FE FF &gt;</td>
 </tr>
</tbody></table>

At this point UTF-8 uses three bytes to represent the code point.  One may wonder at this point how a code point greater than U+FFFF would be represented in UTF-16.  The answer lies in surrogate pairs, which use two double-byte sequences together.  Consider the code point <span class="uchar">U+10FFFD PRIVATE USE CHARACTER-10FFFD</span> in the following table.

<table>
 <thead><tr>
  <td>UTF Format</td>
  <td>Byte sequence</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>UTF-8</td>
  <td>&lt; F4 8F BF BD &gt;</td>
 </tr>
 <tr>
  <td>UTF-16 Little Endian</td>
  <td>&lt; FF DB &gt; &lt; FD DF &gt;</td>
 </tr>
 <tr>
  <td>UTF-16 Big Endian</td>
  <td>&lt; DB FF &gt; &lt; DF FD &gt;</td>
 </tr>
 <tr>
  <td>UTF-32 Little Endian</td>
  <td>&lt; FD FF 10 00 &gt;</td>
 </tr>
 <tr>
  <td>UTF-32 Big Endian</td>
  <td>&lt; 00 10 FF FD &gt;</td>
 </tr>
</tbody></table>

Surrogate pairs combine two pairs in the reserved code point range U+D800 to U+DFFF, to be capable of representing all of Unicode’s code points in the 16 bit format.  For this reason, UTF-16 is considered a variable-width encoding just as is UTF-8.  UTF-32 however, is considered a fixed-width encoding.

## <a id="escape"></a>Character Escape Sequences and Entity References

An alternative to encoding characters is representing them using a symbolic representation rather than a serialization of bytes.  This is common in HTTP with URL-encoded data, and in HTML.   In HTML, numerical character references (NCR) can be used in either a decimal or hexadecimal form that maps to a Unicode code point. 

In fact, CSS (Cascading Style Sheets) and even Javascript use escape sequences, as do most programming languages.  The details of each protocol’s specification are outside the scope of this document, however examples will be used here for reference.

The following table lists the common escape sequences for <span class="uchar">U+0041 LATIN CAPITAL LETTER A</span>.

<table>
 <thead><tr>
  <td>UTF Format</td>
  <td>Character Reference or Escape Sequence</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>URL</td>
  <td>%41</td>
 </tr>
 <tr>
  <td>NCR (decimal)</td>
  <td>&amp;#65;</td>
 </tr>
 <tr>
  <td>NCR (Hex)</td>
  <td>&amp;#x41;</td>
 </tr>
 <tr>
  <td>CSS</td>
  <td>\41 and \0041</td>
 </tr>
 <tr>
  <td>Javascript</td>
  <td>\x41 and \u0041</td>
 </tr>
 <tr>
  <td>Other</td>
  <td>\u0041</td>
 </tr>
</tbody></table>

The following table gives another example, using <span class="uchar">U+FEFF ZERO WIDTH NO-BREAK SPACE</span>, also known as the Unicode Byte Order Mark. 

<table>
 <thead><tr>
  <td>UTF Format</td>
  <td>Character  Reference or Escape Sequence</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>URL</td>
  <td>%EF%BB%BF</td>
 </tr>
 <tr>
  <td>NCR (decimal)</td>
  <td>&amp;#65279;</td>
 </tr>
 <tr>
  <td>NCR (Hex)</td>
  <td>&amp;#xFEFF;</td>
 </tr>
 <tr>
  <td>CSS</td>
  <td>&nbsp;\FEFF</td>
 </tr>
 <tr>
  <td>Javascript</td>
  <td>\xEF\xBB\xBF and \uFEFF</td>
 </tr>
 <tr>
  <td>Other</td>
  <td>\uFEFF</td>
 </tr>
</tbody></table>

## <a id="testing"></a>Security Testing Focus Areas

This guide has been designed with two general areas in mind - one being to aid readers in setting goals for a software security assessment. Where possible, data has also been provided to assist software engineers in developing more security software. Information such as how framework API's behave by default and when overridden is subject to change at any time.

Clearly, any protocol and standard can be subject to security vulnerabilities, examples include HTML, HTTP, TCP, DNS.  Character encodings and the Unicode standard are also exposed to vulnerability. Sometimes vulnerabilities are related to a design-flaw in the standard, but more often they’re related to implementation in practice. Many of the phenomena discussed here are not vulnerabilities in the standard. Instead, the following general categories of vulnerability are most common in applications which are not built to anticipate and prevent the relevant attacks:

* Visual Spoofing
* Best-fit mappings
* Charset transcodings and character mappings
* Normalization
* Canonicalization of overlong UTF-8
* Over-consumption
* Character substitution
* Character deletion
* Casing
* Buffer overflows
* Controlling Syntax
* Charset mismatches

Consider the following image as an example.  In the case of <span class="uchar">U+017F LATIN SMALL LETTER LONG S</span>, the upper casing and normalization operations transform the character into a completely different value.  Many characters such as this one have explicit mappings defined through the Unicode Standard, indicating what character (or sequences of characters) they should transform to through casing and normalization.  Normalization is a defined process discussed later in this document.  In some situations, this behavior could be exploited to create cross-site scripting or other attack scenarios.

The rest of this guide intends to explore each of these phenomena in more detail, as each relates to software vulnerability mitigation and testing.


