---
title: "List Pagination for YANG-driven Protocols"
abbrev: "List Pagination"
category: std

docname: draft-ietf-netconf-list-pagination-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Network Configuration"
keyword:
 - list pagination
 - yang data model

author:
 -
    fullname:  Kent Watsen
    organization: Watsen Networks
    email: kent+ietf@watsen.net
 -
    fullname:  Qin Wu
    organization: Huawei
    email: bill.wu@huawei.com
 -
    fullname:  Per Andersson
    organization: Cisco
    email: per.ietf@ionio.se
 -
    fullname:  Olof Hagsand
    organization: SUNET
    email: olof@hagsand.se
 -
    fullname:  Hongwei Li
    organization: HP
    email: flycoolman@gmail.com

normative:

informative:

...

--- abstract

   In some circumstances, instances of YANG modeled "list" and "leaf-
   list" nodes may contain numerous entries.  Retrieval of all the
   entries can lead to inefficiencies in the server, the client, and the
   network in between.

   This document defines a model for list pagination that can be
   implemented by YANG-driven management protocols such as NETCONF and
   RESTCONF.  The model supports paging over optionally filtered and/or
   sorted entries.  The solution additionally enables servers to
   constrain query expressions on some "config false" lists or leaf-
   lists.

--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
