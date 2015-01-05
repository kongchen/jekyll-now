---
layout: post
title: LDIF的语法图(铁路图)
date: 2012-11-30 13:32:23.000000000 +08:00
categories:
- 技术
tags:
- ldap
- ldif
status: publish
type: post
published: true
meta:
  _wp_old_slug: ldif%e7%9a%84%e8%af%ad%e6%b3%95%e5%9b%be%e9%93%81%e8%b7%af%e5%9b%be
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1534287569'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
话说，我对当初第一次接触json这事儿印象十分深刻，原因就在于当时在[json.org][0]看到的那些-不知其名的-图：

![](assets/string.gif)

之所以又念叨起它来，源自这两天在看LDIF 的RFC------[The LDAP Data Interchange Format (LDIF) - Technical Specification][1]。其中<Formal Syntax Definition of LDIF\>这个section里用纯文字（by [ABNF][2]）描述了LDIF的语法。描述固然精确(不精确也不能成为RFC了)，但很显然不如json的这种直观。

实在看得太晕，就想找找有没有用相似形式来表达LDIF语法的现成的东西。

Google还是让我没花什么力气地找到了答案，而且还是一箭双雕。在stackoverflow里有一个[这样的帖子][3] 非但告诉了我这个图的学名------[Railroad Diagram (aka. Syntax Diagram) ][4]，还告诉了我有这[么一个在线网站][5]可以生成它！awesome!!!

这个在线工具支持输入EBNF的文本语法描述，输出railroad diagram。我稍微花了点时间将LDIF rfc里的ABNF改成了EBNF，于是就有了这个很直观易懂且支持跳转操作的railroad diagram([!!!点我查看!!!][6])  
附我改写的ENBF:

\[code title="EBNF"\]  
ldif-file ::= ldif-content | ldif-changes  
ldif-content ::= version-spec (SEP+ ldif-attrval-record)+  
ldif-changes ::= version-spec (SEP ldif-change-record)+  
ldif-attrval-record ::= dn-spec SEP attrval-spec+  
ldif-change-record ::= dn-spec SEP (control)\* changerecord

// version-number MUST be "1" for the LDIF format described in this document.  
version-spec ::= "version:" FILL version-number

version-number ::= DIGIT+

dn-spec ::= "dn:" (FILL distinguishedName | ":" FILL base64-distinguishedName)

// distinguished name, as defined in \[3\]  
distinguishedName ::= SAFE-STRING

//a distinguishedName which has been base64 encoded (see note 10, below)  
base64-distinguishedName ::= BASE64-UTF8-STRING

//a relative distinguished name, defined as in \[3\]  
rdn ::= SAFE-STRING

//an rdn which has been base64 encoded (see note 10, below)  
base64-rdn ::= BASE64-UTF8-STRING

//(See note 9, below)  
control ::= "control:" FILL ldap-oid (SPACE+ ("true" | "false"))? (value-spec)? SEP

//An LDAPOID, as defined in \[4\]  
ldap-oid ::= DIGIT+ ("." DIGIT+)?

attrval-spec ::= AttributeDescription value-spec SEP

//See notes 7 and 8, below  
value-spec ::= ":" ( FILL (SAFE-STRING)? | ":" FILL (BASE64-STRING) | "<" FILL URL)

url ::= 'a Uniform Resource Locator, as defined in \[6\]'

//Definition taken from \[4\]  
AttributeDescription ::= AttributeType (";" options)?

AttributeType ::= ldap-oid | (ALPHA (attr-type-chars)\*)

options ::= option | (option ";" options)

option ::= opt-char+

attr-type-chars ::= ALPHA | DIGIT | "-"

opt-char ::= attr-type-chars

changerecord ::= "changetype:" FILL (change-add | change-delete | change-modify | change-moddn)

change-add ::= "add" SEP attrval-spec+

change-delete ::= "delete" SEP

change-moddn ::= ("modrdn" | "moddn") SEP "newrdn:" ( FILL rdn | ":" FILL base64-rdn) SEP "deleteoldrdn:" FILL ("0" | "1") SEP ("newsuperior:"( FILL distinguishedName | ":" FILL base64-distinguishedName) SEP)?

change-modify ::= "modify" SEP mod-spec\*

mod-spec ::= ("add:" | "delete:" | "replace:") FILL AttributeDescription SEP attrval-spec\* "-" SEP

SPACE ::= \#x20

FILL ::= SPACE\*

SEP ::= (CR LF | LF)

CR ::= \#x0D

LF ::= \#x0A

ALPHA ::= \[A-Z\] | \[a-z\]

DIGIT ::= \[0-9\]

UTF8-1 ::= \[\#x80-\#xBF\]

UTF8-2 ::= \[\#xC0-\#xDF\] UTF8-1

UTF8-3 ::= \[\#xE0-\#xEF\] UTF8-1 UTF8-1

UTF8-4 ::= \[\#xF0-\#xF7\] UTF8-1 UTF8-1 UTF8-1

UTF8-5 ::= \[\#xF8-\#xFB\] UTF8-1 UTF8-1 UTF8-1 UTF8-1

UTF8-6 ::= \[\#xFC-\#xFD\] UTF8-1 UTF8-1 UTF8-1 UTF8-1 UTF8-1

/\* any value <= 127 decimal except NUL, LF, and CR\*/  
SAFE-CHAR ::= \[\#x01-\#x09\] | \[\#x0B-\#x0C\] | \[\#x0E-\#x7F\]

/\* any value <= 127 except NUL, LF, CR, SPACE, colon (":", ASCII 58 decimal) and less-than ("<" , ASCII 60 decimal)\*/  
SAFE-INIT-CHAR ::= \[\#x01-\#x09\] | \[\#x0B-\#x0C\] | \[\#x0E-\#x1F\] | \[\#x21-\#x39\] | \#x3B | \[\#x3D-\#x7F\]

SAFE-STRING ::= (SAFE-INIT-CHAR SAFE-CHAR\*)?

UTF8-CHAR ::= SAFE-CHAR | UTF8-2 | UTF8-3 | UTF8-4 | UTF8-5 | UTF8-6

UTF8-STRING ::= UTF8-CHAR\*

/\*MUST be the base64 encoding of a UTF8-STRING\*/  
BASE64-UTF8-STRING ::= BASE64-STRING

/\* +, /, 0-9, =, A-Z, and a-z as specified in \[5\] \*/  
BASE64-CHAR ::= '+' | '|' | \[0-9\] | '=' | \[A-Z\] | \[a-z\]

BASE64-STRING ::= (BASE64-CHAR\*)?  
\[/code\]

> btw: 同样是在stackoverflow的那个帖子里，有人问Douglas Crockford是用什么工具画的json的railroad diagram的，得到的回答是Microsoft Visio.

试想如果json.org一打开直接是一篇ABNF的描述，80%人的下一步动作估计就是关窗口了。

谁说外表不重要的？

[0]: http://www.json.org
[1]: http://tools.ietf.org/rfc/rfc2849.txt
[2]: http://www.ietf.org/rfc/rfc2234.txt
[3]: http://stackoverflow.com/questions/796824/tool-for-generating-railroad-diagram-used-on-json-org
[4]: http://en.wikipedia.org/wiki/Railroad_diagram
[5]: http://railroad.my28msec.com/rr/ui
[6]: /ldif.xhtml