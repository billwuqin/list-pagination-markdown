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

 REST-Dissertation:
   title: Architectural Styles and the Design of Network-based Software Architectures
   target: http://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm
   date: 2000

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

   YANG modeled "list" and "leaf-list" nodes may contain a large number
   of entries.  For instance, there may be thousands of entries in the
   configuration for network interfaces or access control lists.  And
   time-driven logging mechanisms, such as an audit log or a traffic
   log, can contain millions of entries.

   Retrieval of all the entries can lead to inefficiencies in the
   server, the client, and the network in between.  For instance,
   consider the following:

   *  A client may need to filter and/or sort list entries in order to,
      e.g., present the view requested by a user.

   *  A server may need to iterate over many more list entries than
      needed by a client.

   *  A network may need to convey more data than needed by a client.

   Optimal global resource utilization is obtained when clients are able
   to cherry-pick just that which is needed to support the application-
   level business logic.

   This document defines a generic model for list pagination that can be
   implemented by YANG-driven management protocols such as NETCONF
   {{!RFC6241}} and RESTCONF {{!RFC8040}}.  How the NETCONF and RESTCONF
   protocols support list pagination is described in
   {{?I-D.ietf-netconf-list-pagination-nc}} and
   {{?I-D.ietf-netconf-list-pagination-rc}}, respectively.

   The model presented in this document supports paging over optionally
   filtered and/or sorted entries.  Server-side filtering and sorting is
   ideal as servers can leverage indexes maintained by a backend storage
   layer to accelerate queries.


#  Security Considerations for the "ietf-list-pagination" YANG Module

   This section follows the template defined in Section 3.7.1 of
   {{!RFC8407}}.

   The YANG module specified in this document defines a schema for data
   that is designed to be accessed via network management protocols such
   as NETCONF {{!RFC6241}} or RESTCONF {{!RFC8040}}.  The lowest NETCONF layer
   is the secure transport layer, and the mandatory-to-implement secure
   transport is Secure Shell (SSH) {{!RFC6242}}.  The lowest RESTCONF layer
   is HTTPS, and the mandatory-to-implement secure transport is TLS
   {{!RFC8446}}.

   The Network Configuration Access Control Model (NACM) {{!RFC8341}}
   provides the means to restrict access for particular NETCONF or
   RESTCONF users to a preconfigured subset of all available NETCONF or
   RESTCONF protocol operations and content.

   All protocol-accessible data nodes in this module are read-only and
   cannot be modified.  Access control may be configured to avoid
   exposing any read-only data that is defined by the augmenting module
   documentation as being security sensitive.

   Since this module also defines groupings, these considerations are
   primarily for the designers of other modules that use these
   groupings.

   None of the readable data nodes defined in this YANG module are
   considered sensitive or vulnerable in network environments.  The NACM
   "default-deny-all" extension has not been set for any data nodes
   defined in this module.

   This module does not define any RPCs or actions or notifications, and
   thus the security consideration for such is not provided here.

--- back

# Acknowledgments
{:numbered="false"}

   This work has benefited from the discussions of RESTCONF resource
   collection over the years, in particular,
   {{?I-D.ietf-netconf-restconf-collection}} which provides enhanced
   filtering features for the retrieval of data nodes with the GET
   method and {{?I-D.zheng-netconf-fragmentation}} which document large
   size data handling challenge.  The authors would like to thank the
   following for lively discussions on list (ordered by first name):
   Andy Bierman, Martin Björklund, Robert Varga, Rob Wills, and Rob
   Wilton.
