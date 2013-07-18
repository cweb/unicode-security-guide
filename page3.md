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
* [Round Trip](#round-trip) 
* [Best Fit Mappings](#best-fit)
* [Charset Transcoding and Character Mappings](#transcoding) 
* [Normalization](#normalization)
* [Canonicalization of Non-Shortest Form UTF-8](#canonicalization)
* [Over-Consumption](#overconsumption)
  * [Well-formed and Ill-formed Byte Sequences](#formedness)
  * [Handling Ill-formed Byte Sequences](#handling)
* [Handling the Unexpected](#unexpected)
  * [Unexpected Inputs](#unexpected-inputs)
  * [Character Substitution](#unexpected-substitution)
  * [Character Deletion](#unexpected-deletion)
* [Upper and Lower Casing](#casing)
* [Buffer Overflows](#overflows)
  * [Upper and Lower Casing](#overflow-casing)
  * [Normalization](#overflow-normalization)
* [Controlling Syntax](#syntax)
* [Charset Mismatch](#charset)


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
* Character X in the source encoding doesn't exist in the destination encoding, so the software attempts to find a 'best-fit' match.

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

<table>
 <thead><tr>
  <td>Library</td>
  <td>API</td>
  <td>Best-fit default</td>
  <td>Can override</td>
  <td>Guidance</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>.NET 2.0</td>
  <td>System.Text.Encoding</td>
  <td>Yes</td>
  <td>Yes</td>
  <td>Specify EncoderReplacementFallback in the Encoding constructor.</td>
 </tr>
 <tr>
  <td>.NET 3.0</td>
  <td>System.Text.Encoding</td>
  <td>Yes</td>
  <td>Yes</td>
  <td>Specify
  EncoderReplacementFallback in the Encoding constructor.</td>
 </tr>
 <tr>
  <td>.NET 3.0</td>
  <td>DllImport</td>
  <td>Yes</td>
  <td>Yes</td>
  <td>To properly and more safely deal with this, you can use the
  MarshallAsAttribute class to specify a LPWStr type instead of a LPStr.
  [MarshalAs(UnmanagedType.LPWStr)]</td>
 </tr>
 <tr>
  <td>Win32</td>
  <td>WideCharToMultiByte</td>
  <td>Yes</td>
  <td>Yes</td>
  <td>Set the <a href="http://msdn.microsoft.com/en-us/library/dd374130(VS.85).aspx">WC_NO_BEST_FIT_CHARS</a> flag.</td>
 </tr>
 <tr>
  <td>Java</td>
  <td>TBD</td>
  <td></td>
  <td></td>
  <td>...</td>
 </tr>
 <tr>
  <td>ICU</td>
  <td>TBD</td>
  <td></td>
  <td></td>
  <td>...</td>
 </tr>
</tbody></table>

Another important note Shawn Steel tells us on his blog is
that <a href="http://blogs.msdn.com/shawnste/archive/2007/09/24/are-we-going-to-update-or-maintain-the-best-fit-or-code-page-mappings.aspx">Microsoft
does not intend to maintain the best-fit mappings</a>. For these and other
security reasons it's a good idea to avoid best-fit type of behavior.

The following table lists test cases to run from a black-box, external perspective. By interpreting the output/rendered data, a tester can determine if a best-fit conversion may be happening. Note that the mapping tables for best-fit conversions are numerous and large, leading to a nearly insurmountable number of permutations. To top it off, the best-fit behavior varies between vendors, making for an inconsistent playing field that does not lend well to automation. For this reason, focus here will be on data that is known to either normalize or best-fit.  The table below is not comprehensive by any means, and is only being provided with the understanding that something is better than nothing.

<table>
 <thead><tr>
  <td>Target
  char</td>
  <td>Target code point</td>
  <td>Test code point</td>
  <td>Name</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>o</td>
  <td>\u006F</td>
  <td>\u2134</td>
  <td>SCRIPT SMALL O</td>
 </tr>
 <tr>
  <td>o</td>
  <td>\u006F</td>
  <td>\u014D</td>
  <td>LATIN SMALL LETTER O WITH MACRON</td>
 </tr>
 <tr>
  <td>s</td>
  <td>\u0073</td>
  <td>\u017F</td>
  <td>LATIN SMALL LETTER LONG S</td>
 </tr>
 <tr>
  <td>I</td>
  <td>\u0049</td>
  <td>\u0131</td>
  <td>LATIN SMALL
  LETTER DOTLESS I</td>
 </tr>
 <tr>
  <td>i</td>
  <td>\u0069</td>
  <td>\u0129</td>
  <td>LATIN SMALL LETTER I WITH
  TILDE</td>
 </tr>
 <tr>
  <td>K</td>
  <td>\u004B</td>
  <td>\u212A</td>
  <td>KELVIN SIGN</td>
 </tr>
 <tr>
  <td>k</td>
  <td>\u006B</td>
  <td>\u0137</td>
  <td>LATIN SMALL LETTER K WITH CEDILLA</td>
 </tr>
 <tr>
  <td>A</td>
  <td>\u0041</td>
  <td>\uFF21</td>
  <td>FULLWIDTH LATIN CAPITAL LETTER A</td>
 </tr>
 <tr>
  <td>a</td>
  <td>\u0061</td>
  <td>\u03B1</td>
  <td>GREEK SMALL LETTER ALPHA</td>
 </tr>
 <tr>
  <td>"</td>
  <td>\u0022</td>
  <td>\u02BA</td>
  <td>MODIFIER
  LETTER DOUBLE PRIME</td>
 </tr>
 <tr>
  <td>"</td>
  <td>\u0022</td>
  <td>\u030E</td>
  <td>COMBINING DOUBLE VERTICAL LINE
  ABOVE</td>
 </tr>
 <tr>
  <td>"</td>
  <td>\u0027</td>
  <td>\uFF02</td>
  <td>FULLWIDTH QUOTATION MARK</td>
 </tr>
 <tr>
  <td>'</td>
  <td>\u0027</td>
  <td>\u02B9</td>
  <td>MODIFIER LETTER PRIME</td>
 </tr>
 <tr>
  <td>'</td>
  <td>\u0027</td>
  <td>\u030D</td>
  <td>COMBINING VERTICAL LINE ABOVE</td>
 </tr>
 <tr>
  <td>'</td>
  <td>\u0027</td>
  <td>\uFF07</td>
  <td>FULLWIDTH APOSTROPHE</td>
 </tr>
 <tr>
  <td>&lt;</td>
  <td>\u003C</td>
  <td>\uFF1C</td>
  <td>FULLWIDTH LESS-THAN SIGN</td>
 </tr>
 <tr>
  <td>&lt;</td>
  <td>\u003C</td>
  <td>\uFE64</td>
  <td>SMALL LESS-THAN SIGN</td>
 </tr>
 <tr>
  <td>&lt;</td>
  <td>\u003C</td>
  <td>\u2329</td>
  <td>LEFT-POINTING ANGLE BRACKET</td>
 </tr>
 <tr>
  <td>&lt;</td>
  <td>\u003C</td>
  <td>\u3008</td>
  <td>LEFT ANGLE BRACKET</td>
 </tr>
 <tr>
  <td>&lt;</td>
  <td>\u003C</td>
  <td>\u00AB</td>
  <td>LEFT-POINTING
  DOUBLE ANGLE QUOTATION MARK</td>
 </tr>
 <tr>
  <td>&gt;</td>
  <td>\u003E</td>
  <td>\u00BB</td>
  <td>RIGHT-POINTING DOUBLE ANGLE QUOTATION MARK</td>
 </tr>
 <tr>
  <td>&gt;</td>
  <td>\u003E</td>
  <td>\u3009</td>
  <td>RIGHT ANGLE BRACKET</td>
 </tr>
 <tr>
  <td>&gt;</td>
  <td>\u003E</td>
  <td>\u232A</td>
  <td>RIGHT-POINTING ANGLE BRACKET</td>
 </tr>
 <tr>
  <td>&gt;</td>
  <td>\u003E</td>
  <td>\uFE65</td>
  <td>SMALL GREATER-THAN SIGN</td>
 </tr>
 <tr>
  <td>&gt;</td>
  <td>\u003E</td>
  <td>\uFF1E</td>
  <td>FULLWIDTH GREATER-THAN SIGN</td>
 </tr>
 <tr>
  <td>:</td>
  <td>\u003A</td>
  <td>\u2236</td>
  <td>RATIO</td>
 </tr>
 <tr>
  <td>:</td>
  <td>\u003A</td>
  <td>\u0589</td>
  <td>ARMENIAN FULL STOP</td>
 </tr>
 <tr>
  <td>:</td>
  <td>\u003A</td>
  <td>\uFE13</td>
  <td>PRESENTATION FORM FOR VERTICAL COLON</td>
 </tr>
 <tr>
  <td>:</td>
  <td>\u003A</td>
  <td>\uFE55</td>
  <td>SMALL COLON</td>
 </tr>
 <tr>
  <td>:</td>
  <td>\u003A</td>
  <td>\uFF1A</td>
  <td>FULLWIDTH
  COLON</td>
 </tr>
</tbody></table>

These test cases are largely derived from the <a href="http://unicode.org/Public/MAPPINGS/VENDORS/">public best-fit mappings provided by the Unicode Consortium</a>. These are provided to software vendors but do not necessarily they were implemented as documented. In fact, any
software vendor such as Microsoft, IBM, Oracle, can implement these mappings as they desire. 

## <a id="transcoding"></a>Charset Transcoding and Character Mappings

Sometimes characters and strings are transcoded from a source character set into a destination character set. On the surface this phenomena may seem similar to best-fit mappings, but the process is quite different. In general, when software transcodes data from source charset X to destination charset Y, it follows either a data-driven mapping table or an algorithmic formula.

For the most part this process is data-driven. While these tables are standardized somewhere there may be differences between vendors. ICU
maintains a list of its <a href="http://site.icu-project.org/charts/charset">character set mapping tables</a> online. Also, ICU's <a href="http://demo.icu-project.org/icu-bin/convexp">Converter Explorer</a> tool lets you browse the maintained charset mapping tables. 

Data may be transcoded directly from a source charset to a destination charset, however it's also common to use Unicode as the broker. In the latter case the software will first transcode the source charset to Unicode, and from there to the destination charset. Some vendors such as Microsoft are known to leverage the Private Use Area (PUA) when transcoding to Unicode, when a direct mapping cannot be found or when a source byte sequence is invalid or illegal. It's important to be aware of a few pitfalls during the transcoding process.

* When data is transcoded to the PUA, converting it again from the PUA may have unexpected consequences.
* Data can change length, particularly if transcoding to/from a single-byte charset leads to a mult-byte character in the other charset. 

As a software engineer building a mechanism for transcoding data between charsets, it's important to understand these pitfalls and handle these unexpected cases gracefully.

Software vulnerabilities can arise through charset transcodings. To name a few:

* Transcoding data is not always reversible, so data can be irrevocably lost.
* Characters can be manipulated to bypass string handling filters, such as cross-site scripting (XSS) filters, WAF's, and IDS devices.
* Characters can be manipulated to abuse logic in software. For example, characters transcoded into ../ or file:// would prove detrimental in file handling operations. 

## <a id="normalization"></a>Normalization

In Unicode, Normalization of characters and strings follows a specification defined in the <a href="http://unicode.org/reports/tr15/">Unicode Standard Annex #15: Unicode Normalization Forms</a>.  The details of Normalization are not for the faint of heart and will not be discussed in this guide. For engineers and testers, it's at least important to understand that there are four
Normalization forms defined: 

* NFC - Canonical Decomposition
* NFD - Canonical Decomposition, followed by Canonical Composition
* NFKC - Compatibility Decomposition
* NFKD - Compatibility Decomposition,followed by Canonical Composition

When testing for security vulnerabilities, we're often most interested in the <strong>compatibility decomposition forms (NFKC, NFKD)</strong>, but occassionally the canonical decomposition forms will produce interesting transformations as well. Cases where characters, and sequences of characters, transform into something different than the original source, might be used to bypass filters or produce other exploits.  Consider the following image, which depicts the result of normalizing with either NFKC or NFKD for the character <span class="uchar">U+FE64 SMALL LESS-THAN SIGN</span>.

<img class="center" style="max-width: 50%;" src="{{ site.url }}/img/normalization-nfkc-nfkd-003C.png" />

In the above example, the character U+FE64 will transform into U+003C, which might lead to security vulnerability in HTML applications. Consider the next example which shows the result of either NFD or NFKD decomposition applied to the "Turkish I" character <span class="uchar">U+0130 LATIN CAPITAL LETTER I WITH DOT ABOVE</span>.

<img class="center" style="max-width: 60%;" src="{{ site.url }}/img/normalization-turkish-i.png" />

As a software engineer, it becomes evident that Unicode normalization plays an important role, and that it is not always an explicit choice.  Often times normalization is applied implicitly by the underlying framework, platform, or Web browser.  It's important to understand the API's being used directly, and in some cases indirectly (by other processing on the stack). 

### <a id="normalization-apis"></a>Normalization Defaults in Common Libraries
The following table of common library API's lists known behaviors:

<table>
 <thead><tr>
  <td>Library</td>
  <td>API</td>
  <td>Default</td>
  <td>Can override</td>
  <td>Notes</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>.NET</td>
  <td>System.Text.Encoding</td>
  <td>NFC</td>
  <td>Yes</td>
  <td></td>
 </tr>
 <tr>
  <td>Win32</td>
  <td></td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>Java</td>
  <td></td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>ICU</td>
  <td></td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>Ruby</td>
  <td></td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>Python</td>
  <td></td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>PHP</td>
  <td></td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>Perl</td>
  <td></td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
</tbody></table>

### <a id="normalization-browsers"></a>Normalization in Web Browser URLs
The following table captures how Web browsers normalize URLs.  Differences in normalization and character transformations can lead to incompatibility as well as security vulnerability.
<span class="superscript"><a href="http://web.lookout.net/2012/03/unicode-normalization-in-urls.html">source</a></span>

<table>
 <thead><tr>
  <td>Description</td>
  <td>MSIE 9</td>
  <td>FF 5.0</td>
  <td>Chrome 12</td>
  <td>Safari 5</td>
  <td>Opera 11.5</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>Applies normalization in the path</td>
  <td class="green">No</td>
  <td class="green">No</td>
  <td class="green">No</td>
  <td class="red">Yes - NFC</td>
  <td class="green">No</td>
 </tr>
 <tr>
  <td>Applies normalization in the query</td>
  <td class="green">No</td>
  <td class="green">No</td>
  <td class="green">No</td>
  <td class="red">Yes - NFC</td>
  <td class="green">No</td>
 </tr>
 <tr>
  <td>Applies normalization in the fragment</td>
  <td class="green">No</td>
  <td class="green">No</td>
  <td class="red">Yes - NFC</td>
  <td class="red">Yes - NFC</td>
  <td class="green">No</td>
 </tr>
</tbody></table>

### <a id="normalization-test"></a>Normalization Test Cases
The following table lists test cases to run from a black-box, external perspective. By interpreting the output/rendered data, a tester can determine if a normalization transformation may be happening.

<table>
 <thead><tr>
  <td><b>Target char</b></td>
  <td><b>Target code point</b></td>
  <td><b>Test code point</b></td>
  <td><b>Name</b></td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td><b>o</b></td>
  <td>\u006F</td>
  <td>\u2134</td>
  <td>SCRIPT SMALL O</td>
 </tr>
 <tr>
  <td><b>s</b></td>
  <td>\u0073</td>
  <td>\u017F</td>
  <td>LATIN SMALL LETTER LONG S</td>
 </tr>
 <tr>
  <td><b>K</b></td>
  <td>\u004B</td>
  <td>\u212A</td>
  <td>KELVIN SIGN</td>
 </tr>
 <tr>
  <td><b>A</b></td>
  <td>\u0041</td>
  <td>\uFF21</td>
  <td>FULLWIDTH LATIN CAPITAL LETTER A</td>
 </tr>
 <tr>
  <td><b>"</b></td>
  <td>\u0027</td>
  <td>\uFF02</td>
  <td>FULLWIDTH
  QUOTATION MARK</td>
 </tr>
 <tr>
  <td><b>'</b></td>
  <td>\u0027</td>
  <td>\uFF07</td>
  <td>FULLWIDTH APOSTROPHE</td>
 </tr>
 <tr>
  <td><b>&lt;</b></td>
  <td>\u003C</td>
  <td>\uFF1C</td>
  <td>FULLWIDTH
  LESS-THAN SIGN</td>
 </tr>
 <tr>
  <td><b>&lt;</b></td>
  <td>\u003C</td>
  <td>\uFE64</td>
  <td>SMALL LESS-THAN SIGN</td>
 </tr>
 <tr>
  <td><b>&gt;</b></td>
  <td>\u003E</td>
  <td>\uFE65</td>
  <td>SMALL
  GREATER-THAN SIGN</td>
 </tr>
 <tr>
  <td><b>&gt;</b></td>
  <td>\u003E</td>
  <td>\uFF1E</td>
  <td>FULLWIDTH GREATER-THAN SIGN</td>
 </tr>
 <tr>
  <td><b>:</b></td>
  <td>\u003A</td>
  <td>\uFE13</td>
  <td>PRESENTATION FORM FOR VERTICAL COLON</td>
 </tr>
 <tr>
  <td><b>:</b></td>
  <td>\u003A</td>
  <td>\uFE55</td>
  <td>SMALL COLON</td>
 </tr>
 <tr>
  <td><b>:</b></td>
  <td>\u003A</td>
  <td>\uFF1A</td>
  <td>FULLWIDTH COLON</td>
 </tr>
</tbody></table>

TODO If you've determined that input is being normalized but need different characters to exploit the logic, you may use the accompanying test case database.

## <a id="canonicalization"></a>Canonicalization of Non-Shortest Form UTF-8
The UTF-8 encoding algorithm allows for a single code point to be represented in multiple ways. That is, while the Latin letter 'A' is normally represented using the byte 0x22 in UTF-8, it's non-shortest form, or overlong, encoding would be any of the following:

* 0xC1 0x81
* 0xE0 0x81 0x81
* 0xF0 0x80 0x81 0x81
* etc...

Earlier versions of the Unicode Standard applied Postel's law, or, the robustness principle of 'be conservative in what you do, be liberal in what you accept from others.' While the 'generation' of non-shortest form UTF-8 was forbidden, the 'interpretation' of was allowed.  That changed with Unicode Standard version 3.0, when the requirement changed to prohibit both interpretation and generation. In fact, both the 'generation' and 'interpretation' of non-shortest form UTF-8 are currently prohibited by the standard, with one exception - that 'interpretation' only applies to the Basic Multilingual Plane (BMP) code points between U+0000 and U+FFFF. In terms of the common security vulnerabilities discussed in this document, that exception has no bearing, as the ASCII range of characters are not exempt.

Given the history of security vulnerabilities around overlong UTF-8, many frameworks have defaulted to a more secure position of disallowing these forms to be both generated and interpreted. However, it seems that some software still interprets non-shortest form UTF-8 for BMP characters, including ASCII. A common pattern in software follows:

> Process A performs security checks, but does not check for non-shortest forms.

> Process B accepts the byte sequence from process A, and transforms it into UTF-16 while interpreting non-shortest forms.

> The UTF-16 text may then contain characters that should have been filtered out by process A. [source](http://unicode.org/versions/corrigendum1.html)

The overlong form of UTF-8 byte sequences is currently considered an illegal byte sequence. It's therefore a good test case to attempt in software such as Web applications, browsers, and databases.

Some notes about canonicalization and UTF-8 encoded data.

* The ASCII range (0x00 to 0x7F) is preserved in UTF-8.
* UTF-8 can encode any Unicode character U+000000 through U+10FFFF using any number of bytes, thus leading to the non-shortest form problem.
* The Unicode standard (3.0 and later) requires that a code point be serializd in UTF-8 using a byte sequence of one to four bytes in length. [The Corrigendum #1: UTF-8 Shortest](http://unicode.org/versions/corrigendum1.html) Form introduced this conformance requirement.

__Non-shortest form UTF-8__ has been the vector for critical vulnerabilities in the past. From the [Microsoft IIS 4.0 and 5.0 directory traversal vulnerability](http://www.microsoft.com/technet/security/bulletin/MS00-078.mspx) of 2000, which was rediscovered in the product's [WebDAV component in 2009](http://blog.zoller.lu/2009/05/iis-6-webdac-auth-bypass-and-data.html).

Some of the common security vulnerabilities that use non-shortest form UTF-8 as an attack vector include:

* Directory/folder traversal.
* Bypassing folder and file access filters.
* Bypassing HTML and XSS filters.
* Bypassing WAF and NID's type devices.

As a developer trying to protect against this, it becomes important to understand the API's being used directly, and in some cases indirectly (by other processing on the stack). The following table of common library API's lists known behaviors:

<table>
 <thead><tr>
  <td>Library</td>
  <td>API</td>
  <td>Allows non-shortest UTF8</td>
  <td>Can override </td>
  <td>Notes</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>.NET 2.0</td>
  <td>System.Text.Encoding</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>.NET 3.0</td>
  <td>System.Text.Encoding</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>ICU</td>
  <td>System.Text.Encoding</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
</tbody></table>

As a tester/bug hunter looking for the vulnerabilities, the following table lists test cases to run from a black-box, external perspective. The data in this table presents the first few non-shortest forms (__NSF__) UTF-8 as URL encoded data %NN. If you need __raw bytes__ instead, these same hex values apply.   All of the target chars in the first column are ASCII.

<table>
 <thead><tr>
  <td>Target </td>
  <td>NSF 1</td>
  <td>NSF 2</td>
  <td>NSF 3</td>
  <td>Notes</td>
  <td></td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>A</td>
  <td>%C1%81</td>
  <td>%E0%81%81</td>
  <td>%F0%80%81%81</td>
  <td>Latin A useful as a base test case.</td>
  <td></td>
 </tr>
 <tr>
  <td>"</td>
  <td>%C0%A2</td>
  <td>%E0%80%A2</td>
  <td>%F0%80%80%A2</td>
  <td>Double quote</td>
  <td></td>
 </tr>
 <tr>
  <td>'</td>
  <td>%C0%A7</td>
  <td>%E0%80%A7</td>
  <td>%F0%80%80%A7</td>
  <td>Single quote</td>
  <td></td>
 </tr>
 <tr>
  <td>&lt;<o:p></o:p></td>
  <td>%C0%BC</td>
  <td>%E0%80%BC</td>
  <td>%F0%80%80%BC</td>
  <td>Less-than
  sign</td>
  <td></td>
 </tr>
 <tr>
  <td>&gt;<o:p></o:p></td>
  <td>%C0%BE</td>
  <td>%E0%80%BE</td>
  <td>%F0%80%80%BE</td>
  <td>Greater-than sign</td>
  <td></td>
 </tr>
 <tr>
  <td>.</td>
  <td>%C0%AE</td>
  <td>%E0%80%AE</td>
  <td>%F0%80%80%AE</td>
  <td>Full stop </td>
  <td></td>
 </tr>
 <tr>
  <td>/</td>
  <td>%C0%AF</td>
  <td>%E0%80%AF</td>
  <td>%F0%80%80%AF</td>
  <td>Solidus</td>
  <td></td>
 </tr>
 <tr>
  <td>\</td>
  <td>%C1%9C</td>
  <td>%E0%81%9C</td>
  <td>%F0%80%81%9C</td>
  <td>Reverse
  solidus</td>
  <td></td>
 </tr>
</tbody></table>



## <a id="overconsumption"></a>Over-consumption

The Unicode Transformation Formats (e.g. UTF-8 and UTF-16) serialize code points into legal, or well-formed, byte sequences, also called code units. For example, consider the following code points and their corresponding well-formed code units in UTF-8 format.

<table>
 <thead><tr>
  <td>Code
  point</td>
  <td>Description</td>
  <td>UTF-8
  byte sequence</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>U+0041</td>
  <td>LATIN CAPITAL LETTER A</td>
  <td>0x41</td>
 </tr>
 <tr>
  <td>U+FF21</td>
  <td>FULLWIDTH LATIN CAPITAL LETTER A</td>
  <td>0xEC 0xBC 0xA1</td>
 </tr>
 <tr>
  <td>U+00C0</td>
  <td>LATIN CAPITAL LETTER A WITH GRAVE</td>
  <td>0xC3 0x80</td>
 </tr>
</tbody></table>

And following are the same code points in their corresponding well-formed UTF-16 (little endian) format.

<table>
 <thead><tr>
  <td>Code point</td>
  <td>Description</td>
  <td>UTF-16LE byte sequence</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>U+0041</td>
  <td>LATIN CAPITAL LETTER A</td>
  <td>0x00 0x41</td>
 </tr>
 <tr>
  <td>U+FF21</td>
  <td>FULLWIDTH LATIN CAPITAL LETTER A</td>
  <td>0xFF 0x21</td>
 </tr>
 <tr>
  <td>U+00C0</td>
  <td>LATIN CAPITAL LETTER A WITH GRAVE</td>
  <td>0x00 0xC0</td>
 </tr>
</tbody></table>

### <a id="formedness"></a>Well-formed and Ill-formed Byte Sequences
Consider a UTF-8 decoder consuming a stream of data from a file. It encounters a well-formed byte sequence like:

&lt41 C3 80 41&gt;

This sequence is made up of three well-formed _sub-sequences_.  First is the &lt;41&gt;, second is the &lt;C3 80&gt;, and third is the &lt;41&gt;. The second subsequence &lt;C3 80&gt; is a two-byte sequence. The lead byte C3 indicates a two-byte sequence, and the trailing byte 80 is a valid trailing byte. The table below indicates these relationahips. Now consider that the UTF-8 decoder encounters an __ill-formed byte sequence__:

&lt41 C2 C3 80 41&gt;

Taken apart, there are three minimally well-formed subsequences &lt;41&gt;, &lt;C3 80&gt;, and &lt;41&gt;. However, the &lt;C2&gt; is ill-formed because it doesn't have a valid trailing byte, which would be required per the table below. 

<table>
 <thead>
  <tr>
   <td>Code point</td>
   <td>First byte</td>
   <td>Second byte</td>
   <td>Third byte</td>
   <td>Fourth byte</td>
  </tr>
 </thead>
 <tbody><tr>
  <td>U+0000..U+007F</td>
  <td>00..7F</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>U+0080..U+07FF</td>
  <td>C2..DF</td>
  <td>80..BF</td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>U+0800..U+0FFF</td>
  <td>E0</td>
  <td>A0..BF</td>
  <td>80..BF</td>
  <td></td>
 </tr>
 <tr>
  <td>U+1000..U+CFFF</td>
  <td>E1..EC</td>
  <td>80..BF</td>
  <td>80..BF</td>
  <td></td>
 </tr>
 <tr>
  <td>U+D000..U+D7FF</td>
  <td>ED</td>
  <td>80..9F</td>
  <td>80..BF</td>
  <td></td>
 </tr>
 <tr>
  <td>U+E000..U+FFFF</td>
  <td>EE..EF</td>
  <td>80..BF</td>
  <td>80..BF</td>
  <td></td>
 </tr>
 <tr>
  <td>U+10000..U+3FFFF</td>
  <td>F0</td>
  <td>90..BF</td>
  <td>80..BF</td>
  <td>80..BF</td>
 </tr>
 <tr>
  <td>U+40000..U+FFFFF</td>
  <td>F1..F3</td>
  <td>80..BF</td>
  <td>80..BF</td>
  <td>80..BF</td>
 </tr>
 <tr>
  <td>U+100000..U+10FFFF</td>
  <td>F4</td>
  <td>80..BF</td>
  <td>80..BF</td>
  <td>80..BF<a href="http://unicode.org/versions/Unicode5.0.0/ch03.pdf"><sup>source</sup></a></td>
 </tr>
</tbody></table>

The table above shows the legal and valid UTF-8 byte sequences, as defined by the Unicode Standard 5.0. The lower ASCII range 00..7F has always been preserved in UTF-8. Multi-byte sequences start at code point U+0080 and continue from two to four bytes. For example, code point U+0700 would be encoded in UTF-8 as a two byte sequence, with the lead byte somewhere in the range of C2..DF.


### <a id="handling"></a>Handling Ill-formed Byte Sequences
Over-consumption of well-formed byte sequences has been the vector for critical vulnerabilities. These generally expose widespread issues when they affect a widely used library. One example can be found in the [Internationalization Components for Unicode (ICU)](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-0153) in 2009, which would leave almost any Web-application exposed to cross-site scripting (XSS) threats since software such as Apple's Safari Web browser exposed the flaw. Even Web-applications with strong HTML/XSS filters can be vulnerable when the Web browser is non-conformant.

The following input illustrates the over-consumption attack vector, where an attacker controls the <span class="uchar">img</span> element's src attribute, followed by a text fragment in the HTML. The [0xC2] represents the attacker's UTF-8 lead byte with an invalid trailing byte, the double quote " which gets consumed in the resultant string. The HTML text portion including the <span class="uchar">onerror</span> text is also attacker-controlled input. The entire payload becomes:

<span class="indent">&lt;img src="#[0xC2]"&gt; " onerror="alert(1)"&lt;/ br&gt;</span>

The resultant string after over-consumption:

<span class="indent">&lt;img src="#&gt; " onerror="alert(1)"&lt;/ br&gt;</span>

Although the above is a broken fragment of HTML because the <span class="uchar">img</span> element is not properly closed, most browsers will render it as an img element with an <span class="uchar">onerror</span> event handler.

Some of the common security vulnerabilities that exploit an over-consumption flaw as an attack vector include:

* Bypassing folder and file access filters.
* Bypassing parser-based filters such as HTML and XSS filters.
* Bypassing detection signatures in WAF and NID's type devices.

As a developer trying to protect against this, it again becomes important to understand how the API's being used will handle ill-formed byte sequences. The following table of common library API's lists known behaviors:

<table>
 <thead><tr>
  <td>Library</td>
  <td>API</td>
  <td>Allows ill-formed UTF8</td>
  <td>Can override </td>
  <td>Notes</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>.NET 2.0</td>
  <td>System.Text.Encoding</td>
  <td>No</td>
  <td>No</td>
  <td></td>
 </tr>
 <tr>
  <td>.NET 3.0</td>
  <td>UTF8Encoding</td>
  <td>No</td>
  <td>No</td>
  <td></td>
 </tr>
 <tr>
  <td>ICU</td>
  <td>System.Text.Encoding</td>
  <td>Yes</td>
  <td>Yes</td>
  <td></td>
 </tr>
</tbody></table>

As a tester/bug hunter looking for the vulnerabilities, the following table lists test cases to run from a black-box, external perspective. The data in this table presents byte sequences that could elicit __over-consumption__. You can substitute a % before each byte value __to create a URL-encoded value__ for use in testing. This would be applicable for passing ill-formed byte sequences in a Web-application.

<table>
 <thead><tr>
  <td>Source bytes</td>
  <td>Expected safe result</td>
  <td>Desired unsafe result</td>
  <td>Notes</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>C2 22 3C</td>
  <td>22 3C</td>
  <td>3C</td>
  <td>Error handling of C2 overconsumed the trailing 22.</td>
 </tr>
 <tr>
  <td>"</td>
  <td>%C0%A2</td>
  <td>%E0%80%A2</td>
  <td>Double quote</td>
 </tr>
</tbody></table>

Over-consumption typically happens at a layer lower than most developers work at. It's more likely to be in the frameworks, the browsers, the database, etc. If designing a character set or Unicode layer, be sure to include an error condition for cases where valid lead bytes are followed by invalid trailing bytes.

## <a id="unexpected"></a>Handling the Unexpected
Through error handling, filtering, or other cases of input validation, problematic characters or raw bytes might be replaced or deleted. In these cases, it's important that the resultant string or byte sequence does not introduce a vulnerability. This problem is not specific to Unicode by any means, and can occur with any character set. However as will be discussed, Unicode has a good solution.

### <a id="unexpected-input"></a>Unexpected Inputs
TODO
#### Unassigned Code Points
U+2073
#### Illegal Code Points
e.g. half of a surrogate pair

### <a id="unexpected-substitution"></a>Character Substitution
The following input illustrates a dangerous character substitution. In this case, the application uses input validation to detect when a string contains characters such as <span class="uchar">&lt;</span> and then sanitizes such character’s by replacing them with a <span class="uchar">.</span> period, or full stop. Internally, the application fetches files from a file share in the form:

<span class="indent">file://sharename/protected/user-01/files</span>

By exploiting the character substitution logic, an attacker could perform directory traversal attacks on the application:

<span class="indent">file://sharename/protected/user-01/../user-002/files</span>

### <a id="unexpected-detetion"></a>Character Deletion
An application may choose to delete characters when invalid, illegal, or unexpected data is encountered. This can also be problematic if not handled carefully. In general, it's safer to replace with Unicode's <span class="uchar">REPLACEMENT CHARACTER U+FFFD</span> than it is to delete.

Consider a Web-browser that deletes certain special characters such as a mid-stream Unicode BOM when encountered in its HTML parsing. An attacker injects the following HTML which includes the Unicode BOM represented by <span class="uchar">U+FEFF</span>. The existence of this character allows the attacker's input to bypass the Web-application's cross-site scripting filter, which rejects an occurrence of <span class="uchar">&lt;script&gt;</span>.

<span class="indent">&lt;scr[U+FEFF]ipt&gt;</span>

The Unicode BOM has special meaning in the standard, and in most software.  The following image illustrates some of the special properties associated with this character:

TODO add image

The Unicode BOM is recommend input for most software test cases, and can be especially useful when test text parsers such as HTML and XML.

#### Guidance

Handle error conditions securely by replacing with the Unicode <span class="uchar">REPACEMENT CHARACTER U+FFFD</span>. If that's impractical for some reason then choose a safe replacement that doesn't have syntactical meaning in the protocol being used. Some common examples include ? and #.

## <a id="casing"></a>Upper and Lower Casing
Strings are transformed through upper and lower casing operations, and sometimes in ways that weren't intended.  This behavior can be exploited if performed at the wrong time.  For example, if a casing operation is performed anywhere in the stack after a security check, then a special character like <span class="uchar">U+0130 LATIN CAPITAL LETTER I WITH DOT ABOVE</span> could be used to bypass a cross-site scripting filter.

<span class="indent">toLower("&#x0130") == "i"</span>

Another aspect of casing operations is that the length of characters and strings can change, depending on the input.  The following should never be assumed:

<span class="indent">toLower("scr&#x0130pt") == "script"</span>

Another aspect of casing operations is that the length of characters and strings can change, depending on the input.  The following should never be assumed:

<span class="indent">len(x) != len(toLower(x))</span>

Common frameworks handle string comparison in different ways.  The following table captures the behavior of classes intended for case-sensitive and case-insensitive string comparison.

<table>
 <thead><tr>
  <td>Library</td>
  <td>API</td>
  <td>Is Dangerous </td>
  <td>Can override </td>
  <td>Notes</td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>.NET 1.0</td>
  <td>StringComparer</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>.NET 2.0</td>
  <td>StringComparer</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>.NET 3.0</td>
  <td>StringComparer</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>Win32</td>
  <td>CompareStringOrdinal</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>Win32</td>
  <td><a href="http://msdn.microsoft.com/en-us/library/ms647489(VS.85).aspx">lstrcmpi</a></td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>Win32</td>
  <td>CompareStringEx</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>ICU C</td>
  <td>ucol_strcoll</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>ICU C</td>
  <td>ucol_strcollIter</td>
  <td></td>
  <td>Allows for comparing two strings that are supplied as character
  iterators (UCharIterator). This is useful when you need to compare
  differently encoded strings using strcoll</td>
  <td></td>
 </tr>
 <tr>
  <td>ICU C++</td>
  <td>Collator::Compare</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>ICU C</td>
  <td>u_strCaseCompare</td>
  <td></td>
  <td>Compare two strings case-insensitively using full case folding.</td>
  <td></td>
 </tr>
 <tr>
  <td>ICU C</td>
  <td>u_strcasecmp</td>
  <td></td>
  <td>Compare two strings case-insensitively using full
  case folding.</td>
  <td></td>
 </tr>
 <tr>
  <td>ICU C</td>
  <td>u_strncasecmp</td>
  <td></td>
  <td>Compare two strings case-insensitively using full case folding.</td>
  <td></td>
 </tr>
 <tr>
  <td>ICU Java</td>
  <td>caseCompare</td>
  <td></td>
  <td>Compare two strings case-insensitively using full
  case folding.</td>
  <td></td>
 </tr>
 <tr>
  <td></td>
  <td></td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>ICU Java</td>
  <td>Collator.compare</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
 <tr>
  <td>POSIX</td>
  <td>strcoll</td>
  <td></td>
  <td></td>
  <td></td>
 </tr>
</tbody></table>



## <a id="overflows"></a>Buffer Overflows
Buffer overflows can occur through improper assumptions about characters versus bytes, and also about string sizes after casing and normalization operations.

### <a id="overflow-casing"></a>Upper and Lower Casing
The following table from UTR 36 illustrates the maximum expansion factors for casing operations on the edge-case characters in Unicode.  These inputs make excellent test cases.

<table>
 <thead><tr>
  <td>Operation </td>
  <td>UTF </td>
  <td>Factor </td>
  <td>Sample </td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>Lower </td>
  <td>8</td>
  <td>1.5</td>
  <td>&#x023A;</td>
  <td>U+023A</td>
 </tr>
 <tr>
  <td>16, 32</td>
  <td>1</td>
  <td>A</td>
  <td>U+0041</td>
 </tr>
 <tr>
  <td>Upper </td>
  <td>8, 16, 32 </td>
  <td>3 </td>
  <td>&#x0390;</td>
  <td>U+0390</td>
 </tr>
</tbody></table>

<sup>[source:  Unicode Technical Report #36](http://www.unicode.org/reports/tr36/)</sup>

### <a id="overflow-normalization"></a>Normalization

The following table from UTR 36 illustrates the maximum expansion factors for normalization operations on the edge case characters in Unicode.  These inputs make excellent test cases.

<table>
 <thead><tr>
  <td>Operation </td>
  <td>UTF </td>
  <td>Factor </td>
  <td>Sample </td>
 </tr>
 </thead>
 <tbody>
 <tr>
  <td>NFC</td>
  <td>8</td>
  <td>3X</td>
  <td>&#x1D160;</td>
  <td>U+1D160</td>
 </tr>
 <tr>
  <td>16, 32</td>
  <td>3X</td>
  <td>&#xFB2C;</td>
  <td>U+FB2C</td>
 </tr>
 <tr>
  <td>NFD</td>
  <td>8</td>
  <td>3X</td>
  <td>&#x0390;</td>
  <td>U+0390</td>
 </tr>
 <tr>
  <td>16, 32</td>
  <td>4X</td>
  <td>&#x1F82;</td>
  <td>U+1F82</td>
 </tr>
 <tr>
  <td>NFKC/NFKD</td>
  <td>8</td>
  <td>11X</td>
  <td>&#xFDFA;</td>
  <td>U+FDFA</td>
 </tr>
 <tr>
  <td>16, 32</td>
  <td>18X</td>
 </tr>
</tbody></table>
<sup>[source:  Unicode Technical Report #36](http://www.unicode.org/reports/tr36/)</sup>



## <a id="syntax"></a>Controlling Syntax

White space and line feeds affect syntax in parsers such as HTML, XML and javascript.  By interpreting characters such as the 'Ogham space mark' and 'Mongolian vowel separator' as whitespace software can allow attacks through the system.  This could give attackers control over the parser, and enable attacks that might bypass security filters.  Several characters in Unicode are assigned the 'white space' category and also the 'white space' binary property.  Depending on how software is designed, these characters may literally be treated as a space character U+0020.

For example, the following illustration shows the special white space properties associated with the <span class="uchar">U+180E MONGOLIAN VOWEL SEPARATOR</span> character.

TODO: add image

If a Web browser interprets this character as white space U+0020, then the following HTML fragment would execute script:

<span class="indent">&lt;a href=#[U+180E]onclick=alert()&gt;</span>


## <a id="charset"></a>Charset Mismatch

When software cannot accurately determine the character set of the text it is dealing with, then it must decide to either error or make an assumption.  User-agents most commonly must deal with this problem, as they’re faced with interpreting data from a large assortment of character sets.  There are no standards that define how to handle situations of character set mismatch, and vendor implementations vary greatly.

 Consider the following diagram, in which a Web browser receives an HTTP response with an HTTP charset of ISO-8859-1 defined, and a meta tag charset of shift_jis defined in the HTML.

TODO add image

When an attacker can exploit can control charset declarations, they can control the software’s behavior and in some cases setup an attack.
