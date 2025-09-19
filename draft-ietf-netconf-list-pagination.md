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

## Terminology

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
   "OPTIONAL" in this document are to be interpreted as described in BCP
   14{{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all
   capitals, as shown here.

   The following terms are defined in {{!RFC7950}} and are not redefined
   here: client, data model, data tree, feature, extension, module,
   leaf, leaf-list, and server.

## Convention

   Various examples in this document use "BASE64VALUE=" as a placeholder
   value for binary data that has been base64 encoded (per Section 9.8
   of{{!RFC7950}}).  This placeholder value is used because real base64
   encoded structures are often many lines long and hence distracting to
   the example being presented.

## Adherence to the NMDA

   This document is compliant with the Network Management Datastore
   Architecture (NMDA) {{!RFC8342}}.  The "ietf-list-pagination" module
   only defines a YANG extension and augments a couple leafs into a
   "config false" node defined by the "ietf-system-capabilities" module.

#  Solution Overview

   The solution presented in this document broadly entails a client
   sending a query to a server targeting a specific list or leaf-list
   including optional parameters guiding which entries should be
   returned.

   Furthermore the solution is intended to leverage underlying database
   system capabilities, e.g. fast lookups relying on indexes, using
   efficient built-in selection and pagination functions.  However,
   since there are many different database systems and configurations,
   the solution defines a common subset of functionality broadly
   available.  It is also possible that a datastore's underlying
   database system is federated, i.e. consists of several different
   database systems to provide one datastore.  In order to form a
   general solution, the possibility to configure and tune what
   components are available is presented.

   A secondary aspect of this solution entails a client sending a query
   parameter to a server guiding how descendent lists and leaf-lists
   should be returned.  This parameter may be used on any target node,
   not just "list" and "leaf-list" nodes.

   Clients detect a server's support for list pagination via an entry
   for the "ietf-list-pagination" module (defined in Section 4) in the
   server's YANG Library {{!RFC8525}} response.

   Relying on client-provided query parameters ensures servers remain
   backward compatible with legacy clients.

#  Solution Details

   This section is composed of the following subsections:

   *  Section 3.1 defines five query parameters clients may use to page
      through the entries of a single list or leaf-list in a data tree.

   *  Section 3.2 defines one query parameter that clients may use to
      affect the content returned for descendant lists and leaf-lists.

   *  Section 3.3 defines per schema-node tags enabling servers to
      indicate which "config false" lists are constrained and how they
      may be interacted with.

##  Query Parameters for a Targeted List or Leaf-List

   The five query parameters presented in this section are listed in
   processing order.  This processing order is logical, efficient, and
   matches the processing order implemented by database systems, such as
   SQL.

   The order is as follows: a server first processes the "where"
   parameter (see Section 3.1.1), then the "sort-by" parameter (see
   Section 3.1.2), then a combination of the "direction" parameter (see
   Section 3.1.4), with either the "offset" parameter (see
   Section 3.1.5) or the "cursor" parameter (see Section 3.1.6), and
   lastly the "limit" parameter (see Section 3.1.7).

   The sorting can furthermore be configured with a locale for sorting.
   This is done by setting the "locale" parameter (see Section 3.1.3).

   The feature "sort" is used to indicate support for the query
   parameters "locale" and "sort-by".

###  The "where" Query Parameter

   Description
      The "where" query parameter specifies a filter expression that
      result-set entries must match.

   Default Value
      If this query parameter is unspecified, then no entries are
      filtered from the working result-set.

   Allowed Values
      The allowed values are XPath 1.0 expressions.  The XPath context
      follows Section 6.4.1 of {{!RFC7950}} and details (such as prefix
      bindings) are further defined at the protocol level, see
      {{?I-D.ietf-netconf-list-pagination-nc}} and
      {{?I-D.ietf-netconf-list-pagination-rc}}.  It is an error if the
      XPath expression references a node identifier that does not exist
      in the schema, is optional or conditional in the schema or, for
      constrained "config false" lists and leaf-lists (see Section 3.3),
      if the node identifier does not point to a node having the
      "indexed" extension statement applied to it (see Section 3.3.2).

   Conformance
      The "where" query parameter MUST be supported for all "config
      true" lists and leaf-lists and SHOULD be supported for "config
      false" lists and leaf-lists.  The supplied expression is allowed
      to be of arbitrary complexity.  Servers MAY disable the support
      for some or all "config false" lists and leaf-lists as described
      in Section 3.3.2.

###  The "sort-by" Query Parameter

   Description
      The "sort-by" query parameter indicates the node in the working
      result-set (i.e., after the "where" parameter has been applied)
      that entries should be sorted by.  Sorts are in ascending order
      (e.g., '1' before '9', 'a' before 'z', etc.).  Missing values are
      sorted to the end (e.g., after all nodes having values).  Sub-
      sorts are not supported.

   Default Value
      If this query parameter is unspecified, then the list or leaf-
      list's default order is used, per the YANG "ordered-by" statement
      (see Section 7.7.7 of {{!RFC7950}}).

   Allowed Values
      The allowed values are node identifiers.  Clients are allowed to
      request sorting by any data node within the result-set.  It is an
      error if the specified node identifier does not exist in the
      schema, is optional or conditional in the schema or, for
      constrained "config false" lists and leaf-lists (see Section 3.3),
      if the node identifier does not point to a node having the
      "indexed" extension statement applied to it (see Section 3.3.2).

   Conformance
      The "sort-by" query parameter MUST be supported for all "config
      true" lists and leaf-lists and SHOULD be supported for "config
      false" lists and leaf-lists.  Servers MAY disable the support for
      some or all "config false" lists and leaf-lists as described in
      Section 3.3.2.

###  The "locale" Query Parameter

   Description
      The "locale" query parameter indicates what locale is used when
      sorting the result-set.  Note that the "locale" query parameter is
      invalid to supply without also supplying the "sort-by" query
      parameter.  If a query supplies "locale" and not "sort-by", error-
      type application and error-tag "invalid-value" is returned.

   Default Value
      If this query parameter is unspecified, it is up to the server to
      select a locale for sorts.  How the server chooses the locale used
      is out of scope for this document.  The result-set includes the
      locale used by the server for sorts with a metadata value
      {{!RFC7952}} called "locale".

   Allowed Values
      The format is a free form string but SHOULD follow the language
      sub-tag format defined in {{!RFC5646}}.  An example is 'sv_SE'.  If a
      supplied locale is unknown to the server, the "locale-unavailable"
      SHOULD be produced in the error-app-tag in the error output.  Note
      that all locales are assumed to be UTF-8, since character encoding
      for YANG strings and all known YANG modelled encodings and
      protocols are required to be UTF-8{{!RFC6241}} {{!RFC7950}} {{!RFC7951}}
      {{!RFC8040}}.  A server MUST accept a known encoding with or without
      trailing ".UTF-8" and MAY emit an encoding with or without
      trailing ".UTF-8".  This means a server must handle both e.g.
      "sv_SE" and "sv_SE.UTF-8" equally as input, and chooses how to
      emit used locale as output.

   Conformance
      The "locale" query parameter MUST be supported for all "config
      true" lists and leaf-lists and SHOULD be supported for "config
      false" lists and leaf-lists.  Servers MAY disable the support for
      some or all "config false" lists and leaf-lists as described in
      Section 3.3.2.

###  The "direction" Query Parameter

   Description
      The "direction" query parameter indicates how the entries in the
      working result-set (i.e., after the "sort-by" parameter has been
      applied) should be traversed.

   Default Value
      If this query parameter is unspecified, the default value is
      "forwards".

   Allowed Values
      The allowed values are:

      forwards
         Return entries in the forwards direction.  Also known as the
         "default" or "ascending" direction.

      backwards
         Return entries in the backwards direction.  Also known as the
         "reverse" or "descending" direction

   Conformance
      The "direction" query parameter MUST be supported for all lists
      and leaf-lists.

###  The "offset" Query Parameter

   Description
      The "offset" query parameter indicates the number of entries in
      the working result-set (i.e., after the "direction" parameter has
      been applied) that should be skipped over when preparing the
      response.

   Default Value
      If this query parameter is unspecified, then no entries in the
      result-set are skipped, the same as when the offset value '0' is
      specified.

   Allowed Values
      The allowed values are unsigned integers.  It is an error for the
      offset value to exceed the number of entries in the working
      result-set, and the "offset-out-of-range" identity SHOULD be
      produced in the error-app-tag in the error output when this
      occurs.

   Conformance
      The "offset" query parameter MUST be supported for all lists and
      leaf-lists.

###  The "cursor" Query Parameter

   Description
      The "cursor" query parameter indicates where to start the working
      result-set (i.e., after the "direction" parameter has been
      applied), the elements before the cursor are skipped over when
      preparing the response.  Furthermore, a result set constrained
      with the "limit" query parameter includes metadata values
      {{!RFC7952}} called "next" and "previous", which contains cursor
      values to the next and previous result-sets.  These next and
      previous cursor values are opaque index values for the underlying
      system's database, e.g. a key or other information needed to
      efficiently access the selected result-set.  These "next" and
      "previous" metadata values work as Hypermedia as the Engine of
      Application State (HATEOAS) links {{REST-Dissertation}}.  This means
      that the server does not keep any stateful information about the
      "next" and "previous" cursor or the current page.  Due to their
      ephemeral nature, cursor values are never cached.

   Default Value
      If this query parameter is unspecified, then no entries in the
      result-set are skipped.

   Allowed Values
      The allowed values are opaque encoded positions interpreted by the
      server to index an element in the list, e.g. a list key or other
      information to efficiently access the selected result-set.  It is
      an error to supply an unknown cursor value for the working result-
      set, and the "cursor-not-found" identity SHOULD be produced in the
      error-app-tag in the error output when this occurs.

   Conformance
      The "cursor" query parameter MUST be supported for all "config
      true" lists and SHOULD be supported for all "config false" lists.
      It is however optional to support the "cursor" query parameter for
      "config false" lists and the support must be signaled by the
      server per list.

      Servers indicate that they support the "cursor" query parameter
      for a "config false" list node by having the "cursor-supported"
      extension statement applied to it in the "per-node-capabilities"
      node in the "ietf-system-capabilities" model, and set to "true".

      Since leaf-lists might not have any unique values that can be
      indexed, the "cursor" query parameter is not relevant for leaf-
      lists.  Consider the following leaf-list [1,1,2,3,5], which
      contains elements without uniquely indexable values.  It would be
      possible to use the position, but then the solution would be equal
      to using the "offset" query parameter.

###  The "limit" Query Parameter

   Description
      The "limit" query parameter limits the number of entries returned
      from the working result-set (i.e., after the "offset" parameter
      has been applied).  Any list or leaf-list that is limited
      includes, somewhere in its encoding, a metadata value {{!RFC7952}}
      called "remaining", a positive integer indicating the number of
      elements that were not included in the result-set by the "limit"
      operation, or the value "unknown" in case, e.g., the server
      determines that counting would be prohibitively expensive.  If the
      string "unbounded" is supplied, there is no limit imposed on the
      result-set.

   Default Value
      If this query parameter is unspecified, the number of entries that
      may be returned is unbounded.

   Allowed Values
      The allowed values are unsigned 32 bit integers greater than or
      equal to 1, or the string "unbounded".

   Conformance
      The "limit" query parameter MUST be supported for all lists and
      leaf-lists.

##  Query Parameter for Descendant Lists and Leaf-Lists

   Whilst this document primarily regards pagination for a list or leaf-
   list, it begs the question for how descendant lists and leaf-lists
   should be handled, which is addressed by the "sublist-limit" query
   parameter described in this section.

###  The "sublist-limit" Query Parameter

   Description

      The "sublist-limit" parameter limits the number of entries
      returned for descendent lists and leaf-lists.

      Any descendent list or leaf-list limited by the "sublist-limit"
      parameter includes, somewhere in its encoding, a metadata value
      {{!RFC7952}} called "remaining", a positive integer indicating the
      number of elements that were not included by the "sublist-limit"
      parameter, or the value "unknown" in case, e.g., the server
      determines that counting would be prohibitively expensive.

      When used on a list node, it only affects the list's descendant
      nodes, not the list itself, which is only affected by the
      parameters presented in Section 3.1.

   Default Value
      If this query parameter is unspecified, the number of entries that
      may be returned for descendent lists and leaf-lists is unbounded.

   Allowed Values
      The allowed values are positive integers.

   Conformance
      The "sublist-limit" query parameter MUST be supported for all
      conventional nodes, including a datastore's top-level node (i.e.,
      '/').

##  Constraints on "where" and "sort-by" for "config false" Lists

   Some "config false" lists and leaf-lists may contain an enormous
   number of entries.  For instance, a time-driven logging mechanism,
   such as an audit log or a traffic log, can contain millions of
   entries.

   In such cases, "where" and "sort-by" expressions will not perform
   well if the server must bring each entry into memory in order to
   process it.

   The server's best option is to leverage query-optimizing features
   (e.g., indexes) built into the backend database holding the dataset.

   However, translating arbitrary "where" expressions and "sort-by" node
   identifiers into syntax supported by the backend database and/or
   query-optimizers may prove challenging, if not impossible, to
   implement.

   Thusly this section introduces mechanisms whereby a server can:

   1.  Identify which "config false" lists and leaf-lists are
       constrained.

   2.  Identify what node-identifiers and expressions are allowed for
       the constrained lists and leaf-lists.

~~~~
      |  Note: The pagination performance for "config true" lists and
      |  leaf-lists is not considered as servers must already be able to
      |  process them as configuration.  Whilst some "config true' lists
      |  and leaf-lists may contain thousands of entries, they are well
      |  within the capability of server-side processing.
~~~~

###  Identifying Constrained "config false" Lists and Leaf-Lists

   Identification of which lists and leaf-lists are constrained occurs
   in the schema tree, not the data tree.  However, as server abilities
   vary, it is not possible to define constraints in YANG modules
   defining generic data models.

   In order to enable servers to identify which lists and leaf-lists are
   constrained, the solution presented in this document augments the
   data model defined by the "ietf-system-capabilities" module presented
   in {{!RFC9196}}.

   Specifically, the "ietf-list-pagination" module (see Section 4)
   augments a boolean leaf node called "constrained" into the "per-node-
   capabilities" node defined in the "ietf-system-capabilities" module.

   The "constrained" leaf MAY be specified for any "config false" list
   or leaf-list.

   When a list or leaf-list is constrained:

   *  All parts of XPath 1.0 expressions are disabled unless explicitly
      enabled by Section 3.3.2.

   *  Node-identifiers used in "where" expressions and "sort-by" filters
      MUST have the "indexed" leaf applied to it (see Section 3.3.2).

   *  For lists only, node-identifiers used in "where" expressions and
      "sort-by" filters MUST NOT descend past any descendent lists.
      This ensures that only indexes relative to the targeted list are
      used.  Further constraints on node identifiers MAY be applied in
      Section 3.3.2.

###  Indicating the Constraints for "where" Filters and "sort-by"
        Expressions

   This section identifies how constraints for "where" filters and
   "sort-by" expressions are specified.  These constraints are valid
   only if the "constrained" leaf described in the previous section
   Section 3.3.1 has been set on the immediate ancestor "list" node or,
   for "leaf-list" nodes, on itself.

####  Indicating Filterable/Sortable Nodes

   For "where" filters, an unconstrained XPath expressions may use any
   node in comparisons.  However, efficient mappings to backend
   databases may support only a subset of the nodes.

   Similarly, for "sort-by" expressions, efficient sorts may only
   support a subset of the nodes.

   In order to enable servers to identify which nodes may be used in
   comparisons (for both "where" and "sort-by" expressions), the "ietf-
   list-pagination" module (see Section 4) augments a boolean leaf node
   called "indexed" into the "per-node-capabilities" node defined in the
   "ietf-system-capabilities" module (see {{!RFC9196}}).

   When a "list" or "leaf-list" node has the "constrained" leaf, only
   nodes having the "indexed" node may be used in "where" and/or "sort-
   by" expressions.  If no nodes have the "indexed" leaf, when the
   "constrained" leaf is present, then "where" and "sort-by" expressions
   are disabled for that list or leaf-list.

#  The "ietf-list-pagination" Module

   The "ietf-list-pagination" module is used by servers to indicate that
   they support pagination on YANG "list" and "leaf-list" nodes, and to
   provide an ability to indicate which "config false" list and/or
   "leaf-list" nodes are constrained and, if so, which nodes may be used
   in "where" and "sort-by" expressions.

##  Data Model Overview

   The following tree diagram {{!RFC8340}} illustrates the "ietf-list-
   pagination" module:

~~~~
   module: ietf-list-pagination

     augment /sysc:system-capabilities/sysc:datastore-capabilities
               /sysc:per-node-capabilities:
       +--ro constrained?        boolean
       +--ro indexed?            boolean
       +--ro cursor-supported?   boolean
~~~~

   Comments:

   *  As shown, this module augments three optional leafs into the "per-
      node-capabilities" node of the "ietf-system-capabilities" module.

   *  Not shown is that the module also defines an "md:annotation"
      statement named "remaining".  This annotation may be present in a
      server's response to a client request containing either the
      "limit" (Section 3.1.7) or "sublist-limit" parameters
      (Appendix A.3.8).

##  Example Usage

###  Constraining a "config false" list

   The following example illustrates the "ietf-list-pagination" module's
   augmentations of the "system-capabilities" data tree.  This example
   assumes the "example-social" module defined in the Appendix A.1 is
   implemented.

~~~~
   =============== NOTE: '\' line wrapping per RFC 8792 ================
   <system-capabilities
     xmlns="urn:ietf:params:xml:ns:yang:ietf-system-capabilities"
     xmlns:ds="urn:ietf:params:xml:ns:yang:ietf-datastores"
     xmlns:es="https://example.com/ns/example-social"
     xmlns:lpg="urn:ietf:params:xml:ns:yang:ietf-list-pagination">
     <datastore-capabilities>
       <datastore>ds:operational</datastore>
       <per-node-capabilities>
         <node-selector>/es:audit-logs/es:audit-log</node-selector>
         <lpg:constrained>true</lpg:constrained>
       </per-node-capabilities>
       <per-node-capabilities>
         <node-selector>/es:audit-logs/es:audit-log/es:timestamp</node-\
   selector>
         <lpg:indexed>true</lpg:indexed>
       </per-node-capabilities>
       <per-node-capabilities>
         <node-selector>/es:audit-logs/es:audit-log/es:member-id</node-\
   selector>
         <lpg:indexed>true</lpg:indexed>
       </per-node-capabilities>
       <per-node-capabilities>
         <node-selector>/es:audit-logs/es:audit-log/es:outcome</node-se\
   lector>
         <lpg:indexed>true</lpg:indexed>
       </per-node-capabilities>
     </datastore-capabilities>
   </system-capabilities>

~~~~

##  YANG Module

   This YANG module has normative references to {{!RFC7952}} and {{!RFC9196}}.
~~~~
   <CODE BEGINS> file "ietf-list-pagination@2025-04-03.yang"

   module ietf-list-pagination {
     yang-version 1.1;
     namespace
       "urn:ietf:params:xml:ns:yang:ietf-list-pagination";
     prefix lpg;

     import ietf-datastores {
       prefix ds;
       reference
         "RFC 8342: Network Management Datastore Architecture (NMDA)";
     }
     import ietf-yang-types {
       prefix yang;
       reference
         "RFC 6991: Common YANG Data Types";
     }

     import ietf-yang-metadata {
       prefix md;
       reference
         "RFC 7952: Defining and Using Metadata with YANG";
     }

     import ietf-system-capabilities  {
       prefix sysc;
       reference
         "RFC 9196: YANG Modules Describing Capabilities for Systems and
                    Datastore Update Notifications";
     }

     organization
       "IETF NETCONF (Network Configuration) Working Group";

     contact
       "WG Web:   <https://datatracker.ietf.org/wg/netconf/>
        WG List:  <mailto:netconf@ietf.org>

        Author:   Kent Watsen
                  <kent+ietf@watsen.net>

        Author:   Qin Wu
                  <bill.wu@huawei.com>

        Author:   Per Andersson
                  <per.ietf@ionio.se>

        Author:   Olof Hagsand
                  <olof@hagsand.se>

        Author:   Hongwei Li
                  <flycoolman@gmail.com>";

     description
       "This module is used by servers to 1) indicate they support
        pagination on 'list' and 'leaf-list' resources, 2) define a
        grouping for each list-pagination parameter, and 3) indicate
        which 'config false' lists have constrained 'where' and
        'sort-by' parameters and how they may be used, if at all.
        Copyright (c) 2024 IETF Trust and the persons identified
        as authors of the code. All rights reserved.

        Redistribution and use in source and binary forms, with
        or without modification, is permitted pursuant to, and
        subject to the license terms contained in, the Revised
        BSD License set forth in Section 4.c of the IETF Trust's
        Legal Provisions Relating to IETF Documents
        (https://trustee.ietf.org/license-info).

        This version of this YANG module is part of RFC XXXX
        (https://www.rfc-editor.org/info/rfcXXXX); see the RFC
        itself for full legal notices.

        The key words 'MUST', 'MUST NOT', 'REQUIRED', 'SHALL',
        'SHALL NOT', 'SHOULD', 'SHOULD NOT', 'RECOMMENDED',
        'NOT RECOMMENDED', 'MAY', and 'OPTIONAL' in this document
        are to be interpreted as described in BCP 14 (RFC 2119)
        (RFC 8174) when, and only when, they appear in all
        capitals, as shown here.";

     revision 2025-04-03 {
       description
         "Initial revision.";
       reference
         "RFC XXXX: List Pagination for YANG-driven Protocols";
     }

     // Features

     feature sort {
       description
         'This feature indicates that the parameters "locale" and
          "sort-by" are supported.';
     }

     // Annotations

     md:annotation remaining {
       type union {
         type uint32;
         type enumeration {
           enum "unknown" {
             description
               "Indicates that the number of remaining entries is
                unknown to the server in case, e.g., the server has
                determined that counting would be prohibitively
                expensive.";
           }
         }
       }
       description
         "This annotation contains the number of elements not included
          in the result set (a positive value) due to a 'limit' or
          'sublist-limit' operation.  If no elements were removed,
          this annotation MUST NOT appear.  The minimum value (0),
          which never occurs in normal operation, is reserved to
          represent 'unknown'.  The maximum value (2^32-1) is
          reserved to represent any value greater than or equal
          to 2^32-1 elements.";
       reference
         "RFC XXXX Section 3.1.7 The 'limit' Query Parameter";
     }

     md:annotation next {
       type string;
       description
         "This annotation contains the opaque encoded value of the next
          cursor in the pagination.";
       reference
         "RFC XXXX Section 3.1.6 The 'cursor' Query Parameter";
     }

     md:annotation previous {
       type string;
       description
         "This annotation contains the opaque encoded value of the
          previous cursor in the pagination.";
       reference
         "RFC XXXX Section 3.1.6 The 'cursor' Query Parameter";
     }

     md:annotation locale {
       if-feature "sort";
       type string;
       description
         "This annotation contains the locale used when sorting.

          The format is a free form string but SHOULD follow the
          language sub-tag format defined in RFC 5646.
          An example is 'sv_SE'.

          For further details see references:
          RFC 5646: Tags for identifying Languages
          RFC 6365: Technology Used in Internationalization in the
                    IETF";
       reference
         "RFC XXXX Section 3.1.3 The 'locale' Query Parameter";
     }

     // Identities

     identity list-pagination-error {
       description
         "Base identity for list-pagination errors.";
     }

     identity offset-out-of-range {
       base list-pagination-error;
       description
         "The 'offset' query parameter value is greater than the number
          of instances in the target list or leaf-list resource.";
     }

     identity cursor-not-found {
       base list-pagination-error;
       description
         "The 'cursor' query parameter value is unknown for the target
          list.";
     }

     identity locale-unavailable {
       if-feature "sort";
       base list-pagination-error;
       description
         "The 'locale' query parameter input is not a valid
          locale or the locale is not available on the system.";
     }

     // Groupings

     grouping pagination-parameters {
       description
         "This grouping may be used by protocol-specific YANG modules
          to define a protocol-specific pagination parameters.";
       container list-pagination {
         description
           "List pagination parameters.";
         leaf where {
           type union {
             type yang:xpath1.0;
             type enumeration {
               enum "unfiltered" {
                 description
                   "Indicates that no entries are to be filtered
                    from the working result-set.";
               }
             }
           }
           default "unfiltered";
           description
             "The 'where' parameter specifies a boolean expression
              that result-set entries must match.

              It is an error if the XPath expression references a node
              identifier that does not exist in the schema, is optional
              or conditional in the schema or, for constrained 'config
              false' lists and leaf-lists, if the node identifier does
              not point to a node having the 'indexed' extension
              statement applied to it (see RFC XXXX).";
         }
         leaf locale {
           if-feature "sort";
           type string;
           description
             "The 'locale' parameter indicates the locale which the
              entries in the working result-set should be collated.";
         }
         leaf sort-by {
           if-feature "sort";
           type union {
             type string {
               // An RFC 7950 'descendant-schema-nodeid'.
               pattern '([A-Za-z_][A-Za-z0-9_.-]*:)?'
                     + '[A-Za-z_][A-Za-z0-9_.-]*'
                     + '(/[A-Za-z_]([A-Za-z0-9_.-]*:)?'
                     + '[A-Za-z_]([A-Za-z0-9_.-]*))*';
             }
             type enumeration {
               enum "none" {
                 description
                   "Indicates that the list or leaf-list's default
                    order is to be used, per the YANG 'ordered-by'
                    statement.";
               }
             }
           }
           default "none";
           description
             "The 'sort-by' parameter indicates the node in the
              working result-set (i.e., after the 'where' parameter
              has been applied) that entries should be sorted by.
              Sorts are in ascending order (e.g., '1' before '9',
              'a' before 'z', etc.).  Missing values are sorted to
              the end (e.g., after all nodes having values).";
         }
         leaf direction {
           type enumeration {
             enum forwards {
               description
                  "Indicates that entries should be traversed from
                   the first to last item in the working result set.";
             }
             enum backwards {
               description
                  "Indicates that entries should be traversed from
                   the last to first item in the working result set.";
             }
           }
           default "forwards";
           description
             "The 'direction' parameter indicates how the entries in the
              working result-set (i.e., after the 'sort-by' parameter
              has been applied) should be traversed.";
         }
         leaf cursor {
           type string;
           description
             "The 'cursor' parameter indicates where to start the
              working result-set (i.e. after the 'direction' parameter
              has been applied), the elements before the cursor are
              skipped over when preparing the response. Furthermare the
              result-set is annotated with attributes for the next and
              previous cursors following a result-set constrained with
              the 'limit' query parameter.";
         }
         leaf offset {
           type uint32;
           default 0;
           description
             "The 'offset' parameter indicates the number of entries
              in the working result-set (i.e., after the 'direction'
              parameter has been applied) that should be skipped over
              when preparing the response.";
         }
         leaf limit {
           type union {
             type uint32 {
               range "1..max";
             }
             type enumeration {
               enum "unbounded" {
                 description
                   "Indicates that the number of entries that may be
                    returned is unbounded.";
               }
             }
           }
           default "unbounded";
           description
             "The 'limit' parameter limits the number of entries
              returned from the working result-set (i.e., after the
              'offset' parameter has been applied).

              Any result-set that is limited includes, somewhere in its
              encoding, the metadata value 'remaining' to indicate the
              number entries not included in the result set.";
         }
         leaf sublist-limit {
           type union {
             type uint32 {
               range "1..max";
             }
             type enumeration {
               enum "unbounded" {
                 description
                   "Indicates that the number of entries that may be
                    returned is unbounded.";
               }
             }
           }
           default "unbounded";
           description
             "The 'sublist-limit' parameter limits the number of entries
              for descendent lists and leaf-lists.

              Any result-set that is limited includes, somewhere in
              its encoding, the metadata value 'remaining' to indicate
              the number entries not included in the result set.";
         }
       }
     }

     // Protocol-accessible nodes

     augment
       "/sysc:system-capabilities/sysc:datastore-capabilities"
       + "/sysc:per-node-capabilities" {
       // Ensure the following nodes are only used for the
       // <operational> datastore.
       when "/sysc:system-capabilities/sysc:datastore-capabilities"
            + "/sysc:datastore = 'ds:operational'";

       description
         "Defines some leafs that MAY be used by the server to
          describe constraints imposed of the 'where' filters and
          'sort-by' parameters used in list pagination queries.";

       leaf constrained {
         type boolean;
         default false;
         description
           "Indicates that 'where' filters and 'sort-by' parameters
            on the targeted 'config false' list node are constrained.
            If a list is not 'constrained', then full XPath 1.0
            expressions may be used in 'where' filters and all node
            identifiers are usable by 'sort-by'.

            Setting this to true will make the list constrained.

            By default this is false, i.e. a list is not constrained.";
       }
       leaf indexed {
         type boolean;
         default false;
         description
           "Indicates that the targeted descendent node of a
            'constrained' list (see the 'constrained' leaf) may be
            used in 'where' filters and/or 'sort-by' parameters.
            If a descendent node of a 'constrained' list is not
            'indexed', then it MUST NOT be used in 'where' filters
            or 'sort-by' parameters.

            Setting this to true will allow a constrained list to be
            used in 'where' filters and/or 'sort-by' parameters.

            By default this is false, i.e. a list constrained is not
            allowed for 'where' filters or 'sort-by' parameters.";
       }
       leaf cursor-supported {
         type boolean;
         default false;
         description
           "Indicates that the targeted list node supports the
            'cursor' parameter.
            Setting this to true will enable the 'cursor' query
            parameter for a 'config false' list.

            By default this is false, i.e. the 'cursor' query parameter
            is disabled for 'config false' lists.";
       }
     }
   }

   <CODE ENDS>
~~~~

#  IANA Considerations

##  The "IETF XML" Registry

   This document registers one URI in the "ns" subregistry of the IETF
   XML Registry {{!RFC3688}} maintained at
   https://www.iana.org/assignments/xml-registry/xml-registry.xhtml#ns.
   Following the format in {{!RFC3688}}, the following registration is
   requested:

~~~~
           URI: urn:ietf:params:xml:ns:yang:ietf-list-pagination
           Registrant Contact: The IESG.
           XML: N/A, the requested URI is an XML namespace.
~~~~

##  The "YANG Module Names" Registry

   This document registers one YANG module in the YANG Module Names
   registry {{!RFC6020}} maintained at https://www.iana.org/assignments/
   yang-parameters/yang-parameters.xhtml.  Following the format defined
   in {{!RFC6020}}, the below registration is requested:
   
~~~~
        name: ietf-list-pagination
        namespace: urn:ietf:params:xml:ns:yang:ietf-list-pagination
        prefix: lpg
        RFC: XXXX
~~~~

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
