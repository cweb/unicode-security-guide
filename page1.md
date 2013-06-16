---
layout: default
title: Background
permalink: background/
---

# Unicode Security Guide, Part 1, Background #

## Table of Contents 

TODO TOC

# Background 
The Unicode Standard provides a unique number for every character, enabling disparate computing systems to exchange and interpret text in the same way.

## Brief History of Character Encodings 
Early in computing history, it became widely clear that a standardized way to represent characters would provide many benefits. Around 1963, IBM standardized EBCDIC in its mainframes, about the same time that ASCII was standardized as a 7-bit character set.  EBCDIC used an 8-bit encoding, and the unused eighth-bit in ASCII allowed OEM's to apply the extra bit for their proprietary purposes. 

This allowed OEM's to ship computers and later PC's with customized character encodings specific to language or region. So computers could ship to Israel with a tweaked ASCII encoding set that supported Hebrew for example. The divergence in these customized character sets grew into a problem over time, as data interchange become error-prone if not impossible when computers didn't share the same character set. 

In response to this growth, the International Organization for Standardization (ISO) began developing the ISO-8859 set of character encoding standards in the early 1980's. The ISO-8859 standards were aimed at providing a reliable system for data-interchange across computing systems. They provided support for many popular languages, but weren't designed for high-quality typography which needed symbols such as ligatures. 

In the late 1980's Unicode was being designed, around the same time ISO recognized the need for a single character encoding framework, what would later come to be called the Universal Character Set (UCS), or ISO 10646. Version 1.0 of the Unicode standard was released in 1991 at almost the same time as UCS was made public. Since that time, Unicode has become the de facto character encoding model, and has worked closely with ISO and UCS to ensure compatibility and similar goals.

## Brief Introduction to Unicode 
The Unicode framework can presumably represent all of the worlds languages and scripts, past, present, and future. That's because the current version 5.1 of the Unicode Standard has space for over 1 million code points. A code point is a unique value within the Unicode code-space. A single code point can represent a letter, a special control character (e.g. carriage return), a symbol, or even some other abstract thing.

### Code Points 
A code point is a 21-bit scalar value in the current version of Unicode, and is represented using the following type of reference where NNNN would be a hex value: 

> __U+NNNN__

The valid range for Unicode code points is currently U+0000 to U+10FFFF.  This range can be expanded in the future if the Unicode Standard changes. The following image illustrates some of the properties or metadata that accompany a given code point.

Code point U+0041 represents the Latin Capital Letter A. It's no coincidence that this maps directly to ASCII's value 0x41, as the Unicode Standard has always preserved the lower ASCII range to ensure widespread compatibility. Some interesting things to note here are the properties associated with this code point:

* Several categories are assigned including a general 'category' and a 'script' family.
* A 'lower case' mapping is defined.
* An 'upper case' mapping is defined.
* A 'normalization' mapping is defined.
* Binary properties are assigned.

This short list only represents some of the metadata attached to a code point, there can be much more information. In looking for security issues however, this short list provides a good starting point.

## Character Encoding
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
  <td>
  <b><span>UTF
  Format</span></b>
  </td>
  <td>
  <b><span>Byte
  sequence</span></b>
  </td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>
  <b>UTF-8</b>
  </td>
  <td>
  &lt; 41 &gt;
  </td>
 </tr>
 <tr>
  <td>
  <b>UTF-16 Little Endian</b>
  </td>
  <td>
  &lt; 41 00 &gt;
  </td>
 </tr>
 <tr>
  <td>
  <b>UTF-16 Big Endian</b>
  </td>
  <td>
  &lt; 00 41 &gt;
  </td>
 </tr>
 <tr>
  <td>
  <b>UTF-32 Little Endian</b>
  </td>
  <td>
  &lt; 41 00 00 00 &gt;
  </td>
 </tr>
 <tr>
  <td>
  <b>UTF-32 Big Endian</b>
  </td>
  <td>
  &lt; 00 00 00 41 &gt;
  </td>
 </tr>
</tbody></table>

The lower ASCII character set is preserved by UTF-8 up through U+007F.  The following table gives another example, using <span class="uchar">U+FEFF ZERO WIDTH NO-BREAK SPACE</span>, also known as the Unicode Byte Order Mark.

<table>
 <thead><tr>
  <td>
  <p><b><span>UTF
  Format</span></b></p>
  </td>
  <td>
  <p><b><span>Byte
  sequence</span></b></p>
  </td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>
  <p><b>UTF-8</b></p>
  </td>
  <td>
  <p>&lt; EF BB BF &gt;</p>
  </td>
 </tr>
 <tr>
  <td>
  <p><b>UTF-16 Little Endian</b></p>
  </td>
  <td>
  <p>&lt; FF FE &gt;</p>
  </td>
 </tr>
 <tr>
  <td>
  <p><b>UTF-16 Big Endian</b></p>
  </td>
  <td>
  <p>&lt; FE FF &gt;</p>
  </td>
 </tr>
 <tr>
  <td>
  <p><b>UTF-32 Little Endian</b></p>
  </td>
  <td>
  <p>&lt; FF FE 00 00 &gt;</p>
  </td>
 </tr>
 <tr>
  <td>
  <p><b>UTF-32 Big Endian</b></p>
  </td>
  <td>
  <p>&lt; 00 00 FE FF &gt;</p>
  </td>
 </tr>
</tbody></table>
