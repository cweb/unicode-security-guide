---
layout: default
title: Character Transformations
permalink: character-transformations/
---

# Unicode Security Guide
## _Character Transformations_

This section attempts to explore the various ways in which characters and strings can be transformed by software processes.  Such transformations are not vulnerabilities necessarily, but could be exploited by clever attackers. 

As an example, consider an attacker trying to inject script (i.e. cross-site scripting, or XSS attack) into a Web-application which utilizes a defensive input filter.  The attacker finds that the application performs a lowercase operation on the input after filtering, and by injecting special characters they can exploit that behavior.   That is, the string "script" is prevented by the filter, but the string "scr&#x0130;pt" is allowed.

## Table of Contents 
* [Prior Research](#prior) 
* [Attack Scenarios](#attack)
* [Defensive Options](#defense) 

## <a id="round-trip"></a>Round-trip Conversions: A Common Pattern
In practice, Globalized software must be capable of handling many different character sets, and converting data between them. The process for supporting this requirement can generally look like the following:

1. Accept data from any character set, e.g. Unicode, shift_jis, ISO-8859-1.
2. Transform, or convert, data to Unicode for processing and storage.
3. Transform data to original or other character set for output and display.

In this pattern, Unicode is used as the broker. With support for such a large character repetoire, Unicode will often have a character mapping for both sides of this transaction. To illustrate this, consider the following Web application transaction.

1. An application end-user inputs their full name using characters encoded from the shift_jis character set.
2. Before storing in the database, the application converts the user-input to Unicode's UTF-8 format.
3. When visiting the Web page, the user's full name will be returned in UTF-8 format, unless other conditions cause the data to be returned in a different encoding encoding. Such conditions may be based on the Web application's configuration or the user's Web browser language and encoding settings. Under these types of conditions, the Web application will convert the data to the requested encoding.

The round-trip conversions illustrated here can lead to numerous issues that will be further discussed. While it serves as a good example, this isn't the only case where such issues can arise.

## <a id="best-fit"></a>Best-fit Mappings
The "best-fit" phenomena occurs when a character X gets transformed to an entirely different character Y.  This can occur for reasons such as:

* A framework API transforms input to a different character encoding by default.
* Data is marshalled from a wide string type (multi-byte character representation) such as UTF-16, to a non-wide string (single-byte character representation) such as US-ASCII. 
* Character X in the source encoding doesn't exist in the destination encoding, so the software attempts to find a best match.

In general, best-fit mappings occur when characters are transcoded between Unicode and another encoding.  It's often the case that the source encoding is Unicode and the destination is another charset such as shift_jis, however, it could happen in reverse as well. Best-fit mappings are different than character set transcoding which is discussed in another section of this guide.

Software vulnerabilities may arise when best-fit mappings occur. To name a few:

* Best-fit mappings are often not reversible, so data is irrevocably lost.  For example, a common best-fit operation would transform a <span class="uchar">U+FF1C FULLWIDTH LESS-THAN SIGN &#xFF1C;</span> to the  <span class="uchar">U+003C LESS-THAN SIGN</span>, or the ASCII &lt; used in HTML.  Once converted down to the ASCII &lt;, there’s no reliable way to convert back to the FULLWIDTH source.
* Characters can be manipulated to bypass string handling filters, such as cross-site scripting (XSS) filters, WAF's, and IDS devices.
* Characters can be manipulated to abuse logic in software. Such as when the characters can be used to access files on the file system. In this case, a best-fit mapping to characters such as ../ or file:// could be damaging.

For example, consider a Web-application that’s implemented a filter to prevent XSS (cross-site scripting) attacks.  The filter attempts to block most dangerous characters, and operates at an outermost layer in the application.  The implementation might look like:

1. An input validation filter rejects characters such as &lt;, &gt;, ', and " in a Web-application accepting UTF-8 encoded text.
2. An attacker sends in a <span class="uchar">U+FF1C FULLWIDTH LESS-THAN SIGN &#xFF1C;</span> in place of the ASCII &lt;.
3. The attacker’s input looks like:  &#xFF1C;script&gt;
4. After passing through the XSS filter unchanged, the input moves deeper into the application.
5. Another API, perhaps at the data access layer, is configured to use a different character set such as windows-1252. 
6. On receiving the input, a data access layer converts the multi-byte UTF-8 text to the single-byte windows-1252 code page, forcing a best-fit conversion to the dangerous characters the original XSS filter was trying to block.
7.The attacker’s input successfully persists to the database.

[Shawn Steele](http://blogs.msdn.com/shawnste/archive/2006/01/19/515047.aspx) describes the security issues well on his blog, it's a highly recommended short read for the level of coverage he provides regarding Microsoft's API's:

> Best Fit in WideCharToMultiByte and System.Text.Encoding Should be Avoided. Windows and the .Net Framework have the concept of "best-fit" behavior for code pages and encodings. Best fit can be interesting, but often its not a good idea. In WideCharToMultiByte() this behavior is controlled by a WC_NO_BEST_FIT_CHARS flag. In .Net you can use the EncoderFallback to control whether or not to get Best Fit behavior. Unfortunately in both cases best fit is the default behavior. In Microsoft .Net 2.0 best fit is also slower.

As a software engineer, it's important to understand the API's being used directly, and in some cases indirectly (by other processing on the stack). The following table of common library API's lists known behaviors:



## <a id="transcoding"></a>Charset Transcoding and Character Mappings

## <a id="normalization"></a>Normalization

## <a id="canonicalization"></a>Canonicalization of Non-Shortest Form UTF-8

## <a id="overconsumption"></a>Over-consumption
### <a id="formedness"></a>Well-formed and Ill-formed Byte Sequences
### <a id="handling"></a>Handlingg Ill-formed Byte Sequences

## <a id="unexpected"></a>Handling the Unexpected
### <a id="unexpected-input"></a>Unexpected Inputs
### <a id="unexpected-substitution"></a>Character Substitution
### <a id="unexpected-detetion"></a>Character Deletion

## <a id="casing"></a>Upper and Lower Casing

## <a id="overflows"></a>Buffer Overflows
### <a id="overflow-casing"></a>Upper and Lower Casing
### <a id="overflow-normalization"></a>Normalization

## <a id="syntax"></a>Controlling Syntax

## <a id="charset"></a>Charset Mismatch
