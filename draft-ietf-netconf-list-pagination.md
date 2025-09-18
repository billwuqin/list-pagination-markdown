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
   {{?RFC6241}} and RESTCONF {{?RFC8040}}.  How the NETCONF and RESTCONF
   protocols support list pagination is described in
   {{?I-D.ietf-netconf-list-pagination-nc}} and
   {{?I-D.ietf-netconf-list-pagination-rc}}, respectively.

   The model presented in this document supports paging over optionally
   filtered and/or sorted entries.  Server-side filtering and sorting is
   ideal as servers can leverage indexes maintained by a backend storage
   layer to accelerate queries.

## Terminology

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
   "OPTIONAL" in this document are to be interpreted as described in BCP
   14{{?RFC2119}} {{?RFC8174}} when, and only when, they appear in all
   capitals, as shown here.

   The following terms are defined in {{?RFC7950}} and are not redefined
   here: client, data model, data tree, feature, extension, module,
   leaf, leaf-list, and server.

## Convention

   Various examples in this document use "BASE64VALUE=" as a placeholder
   value for binary data that has been base64 encoded (per Section 9.8
   of{{?RFC7950}}).  This placeholder value is used because real base64
   encoded structures are often many lines long and hence distracting to
   the example being presented.

## Adherence to the NMDA

   This document is compliant with the Network Management Datastore
   Architecture (NMDA) {{?RFC8342}}.  The "ietf-list-pagination" module
   only defines a YANG extension and augments a couple leafs into a
   "config false" node defined by the "ietf-system-capabilities" module.

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
