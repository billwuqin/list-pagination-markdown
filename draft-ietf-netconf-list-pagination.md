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

      |  Note: The pagination performance for "config true" lists and
      |  leaf-lists is not considered as servers must already be able to
      |  process them as configuration.  Whilst some "config true' lists
      |  and leaf-lists may contain thousands of entries, they are well
      |  within the capability of server-side processing.

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

###  Indicating the Constraints for "where" Filters and "sort-by" Expressions

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

# The "ietf-list-pagination" Module

The "ietf-list-pagination" module is used by servers to indicate that they support pagination on YANG "list" and "leaf-list" nodes, and to provide an ability to indicate which "config false" list and/or "leaf-list" nodes are constrained and, if so, which nodes may be used in "where" and "sort-by" expressions.

## Data Model Overview
The following tree diagram [RFC8340] illustrates the "ietf-list-pagination" module:

module: ietf-list-pagination

~~~~
  augment /sysc:system-capabilities/sysc:datastore-capabilities
            /sysc:per-node-capabilities:
    +--ro constrained?        boolean
    +--ro indexed?            boolean
    +--ro cursor-supported?   boolean
~~~~

Comments:

As shown, this module augments three optional leafs into the "per-node-capabilities" node of the "ietf-system-capabilities" module.
Not shown is that the module also defines an "md:annotation" statement named "remaining". This annotation may be present in a server's response to a client request containing either the "limit" (Section 3.1.7) or "sublist-limit" parameters (Appendix A.3.8).

## Example Usage

### Constraining a "config false" list
The following example illustrates the "ietf-list-pagination" module's augmentations of the "system-capabilities" data tree. This example assumes the "example-social" module defined in the Appendix A.1 is implemented.

~~~~
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

## YANG Module

~~~~
{::include-fold ./yang/ietf-list-pagination.yang}
~~~~

#  IANA Considerations

##  The "IETF XML" Registry

   This document registers one URI in the "ns" subregistry of the IETF
   XML Registry {{!RFC3688}} maintained at
   https://www.iana.org/assignments/xml-registry/xml-registry.xhtml#ns.
   Following the format in {{!RFC3688}}, the following registration is
   requested:

           URI: urn:ietf:params:xml:ns:yang:ietf-list-pagination
           Registrant Contact: The IESG.
           XML: N/A, the requested URI is an XML namespace.

##  The "YANG Module Names" Registry

   This document registers one YANG module in the YANG Module Names
   registry {{!RFC6020}} maintained at https://www.iana.org/assignments/
   yang-parameters/yang-parameters.xhtml.  Following the format defined
   in {{!RFC6020}}, the below registration is requested:

        name: ietf-list-pagination
        namespace: urn:ietf:params:xml:ns:yang:ietf-list-pagination
        prefix: lpg
        RFC: XXXX

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

# Appendix A. Vector Tests
This normative appendix section illustrates every notable edge condition conceived during this document's production.

Test inputs and outputs are provided in a manner that is both generic and concise.

Management protocol specific documents need only reproduce as many of these tests as necessary to convey peculiarities presented by the protocol.

Implementations are RECOMMENDED to implement the tests presented in this document, in addition to any tests that may be presented in protocol specific documents.

## A.1. Example YANG Module

The vector tests assume the "example-social" YANG module defined in this section.

This module has been specially crafted to cover every notable edge condition, especially with regards to the types of the data nodes.

Following is the tree diagram [RFC8340] for the "example-social" module:

~~~~
module: example-social
  +--rw members
  |  +--rw member* [member-id]
  |     +--rw member-id           string
  |     +--rw email-address       inet:email-address
  |     +--rw password            ianach:crypt-hash
  |     +--rw avatar?             binary
  |     +--rw tagline?            string
  |     +--rw privacy-settings
  |     |  +--rw hide-network?      boolean
  |     |  +--rw post-visibility?   enumeration
  |     +--rw following*          -> /members/member/member-id
  |     +--rw posts
  |     |  +--rw post* [timestamp]
  |     |     +--rw timestamp    yang:date-and-time
  |     |     +--rw title?       string
  |     |     +--rw body         string
  |     +--rw favorites
  |     |  +--rw uint8-numbers*       uint8
  |     |  +--rw uint64-numbers*      uint64
  |     |  +--rw int8-numbers*        int8
  |     |  +--rw int64-numbers*       int64
  |     |  +--rw decimal64-numbers*   decimal64
  |     |  +--rw bits*                bits
  |     +--ro stats
  |        +--ro joined              yang:date-and-time
  |        +--ro membership-level    enumeration
  |        +--ro last-activity?      yang:date-and-time
  +--ro audit-logs
     +--ro audit-log* []
        +--ro timestamp    yang:date-and-time
        +--ro member-id    string
        +--ro source-ip    inet:ip-address
        +--ro request      string
        +--ro outcome      boolean
~~~~

Following is the YANG [RFC7950] for the "example-social" module:

~~~~
{::include-fold ./examples/example-social.yang}
~~~~

## A.2. Example Data Set

The examples assume the server's operational state as follows.

The data is provided in JSON only for convenience and, in particular, has no bearing on the "generic" nature of the tests themselves.

~~~~
{
  "example-social:members": {
    "member": [
      {
        "member-id": "bob",
        "email-address": "bob@example.com",
        "password": "$0$1543",
        "avatar": "BASE64VALUE=",
        "tagline": "Here and now, like never before.",
        "posts": {
          "post": [
            {
              "timestamp": "2020-08-14T03:32:25Z",
              "body": "Just got in."
            },
            {
              "timestamp": "2020-08-14T03:33:55Z",
              "body": "What's new?"
            },
            {
              "timestamp": "2020-08-14T03:34:30Z",
              "body": "I'm bored..."
            }
          ]
        },
        "favorites": {
          "decimal64-numbers": ["3.14159", "2.71828"]
        },
        "stats": {
          "joined": "2020-08-14T03:30:00Z",
          "membership-level": "standard",
          "last-activity": "2020-08-14T03:34:30Z"
        }
      },
      {
        "member-id": "eric",
        "email-address": "eric@example.com",
        "password": "$0$1543",
        "avatar": "BASE64VALUE=",
        "tagline": "Go to bed with dreams; wake up with a purpose.",
        "following": ["alice"],
        "posts": {
          "post": [
            {
              "timestamp": "2020-09-17T18:02:04Z",
              "title": "Son, brother, husband, father",
              "body": "What's your story?"
            }
          ]
        },
        "favorites": {
          "bits": ["two", "one", "zero"]
        },
        "stats": {
          "joined": "2020-09-17T19:38:32Z",
          "membership-level": "pro",
          "last-activity": "2020-09-17T18:02:04Z"
        }
      },
      {
        "member-id": "alice",
        "email-address": "alice@example.com",
        "password": "$0$1543",
        "avatar": "BASE64VALUE=",
        "tagline": "Every day is a new day",
        "privacy-settings": {
          "hide-network": false,
          "post-visibility": "public"
        },
        "following": ["bob", "eric", "lin"],
        "posts": {
          "post": [
            {
              "timestamp": "2020-07-08T13:12:45Z",
              "title": "My first post",
              "body": "Hiya all!"
            },
            {
              "timestamp": "2020-07-09T01:32:23Z",
              "title": "Sleepy...",
              "body": "Catch y'all tomorrow."
            }
          ]
        },
        "favorites": {
          "uint8-numbers": [17, 13, 11, 7, 5, 3],
          "int8-numbers": [-5, -3, -1, 1, 3, 5]
        },
        "stats": {
          "joined": "2020-07-08T12:38:32Z",
          "membership-level": "admin",
          "last-activity": "2021-04-01T02:51:11Z"
        }
      },
      {
        "member-id": "lin",
        "email-address": "lin@users.example.net",
        "password": "$0$1543",
        "privacy-settings": {
          "hide-network": true,
          "post-visibility": "followers-only"
        },
        "following": ["joe", "eric", "alice"],
        "stats": {
          "joined": "2020-07-09T12:38:32Z",
          "membership-level": "standard",
          "last-activity": "2021-04-01T02:51:11Z"
        }
      },
      {
        "member-id": "joe",
        "email-address": "joe@example.com",
        "password": "$0$1543",
        "avatar": "BASE64VALUE=",
        "tagline": "Greatness is measured by courage and heart.",
        "privacy-settings": {
          "post-visibility": "unlisted"
        },
        "following": ["bob"],
        "posts": {
          "post": [
            {
              "timestamp": "2020-10-17T18:02:04Z",
              "body": "What's your status?"
            }
          ]
        },
        "stats": {
          "joined": "2020-10-08T12:38:32Z",
          "membership-level": "pro",
          "last-activity": "2021-04-01T02:51:11Z"
        }
      },
      {
        "member-id": "åsa",
        "email-address": "asa@users.example.net",
        "password": "$0$1543",
        "avatar": "BASE64VALUE=",
        "privacy-settings": {
          "post-visibility": "unlisted"
        },
        "following": ["alice", "bob"],
        "stats": {
          "joined": "2022-02-19T13:12:00Z",
          "membership-level": "standard",
          "last-activity": "2022-04-19T13:12:59Z"
        }
      }
    ]
  },
  "example-social:audit-logs": {
    "audit-log": [
      {
        "timestamp": "2020-10-11T06:47:59Z",
        "member-id": "alice",
        "source-ip": "192.168.0.92",
        "request": "POST /groups/group/2043",
        "outcome": true
      },
      {
        "timestamp": "2020-11-01T15:22:01Z",
        "member-id": "bob",
        "source-ip": "192.168.2.16",
        "request": "POST /groups/group/123",
        "outcome": false
      },
      {
        "timestamp": "2020-12-12T21:00:28Z",
        "member-id": "eric",
        "source-ip": "192.168.254.1",
        "request": "POST /groups/group/10",
        "outcome": true
      },
      {
        "timestamp": "2021-01-03T06:47:59Z",
        "member-id": "alice",
        "source-ip": "192.168.0.92",
        "request": "POST /groups/group/333",
        "outcome": true
      },
      {
        "timestamp": "2021-01-21T10:00:00Z",
        "member-id": "bob",
        "source-ip": "192.168.2.16",
        "request": "POST /groups/group/42",
        "outcome": true
      },
      {
        "timestamp": "2020-02-07T09:06:21Z",
        "member-id": "alice",
        "source-ip": "192.168.0.92",
        "request": "POST /groups/group/1202",
        "outcome": true
      },
      {
        "timestamp": "2020-02-28T02:48:11Z",
        "member-id": "bob",
        "source-ip": "192.168.2.16",
        "request": "POST /groups/group/345",
        "outcome": true
      }
    ]
  }
}
~~~~



## A.3. Example Queries

The following sections are presented in reverse query-parameters processing order. Starting with the simplest (limit) and ending with the most complex (where).

All the vector tests are presented in a protocol-independent manner. JSON is used only for its conciseness.

The "members" list in the example data set is "ordered-by system" and the queries without explicit "sort-by" below return them in this given order. Note that the order of the result-set without explicit "sort-by" is implementation specific for "ordered-by system" lists.

Note that the user "åsa" is only included in Appendix A.3.7. This to isolate the parameter in the example query and its behavior.

### A.3.1. The "limit" Parameter

Noting that "limit" must be a positive number, the edge condition values are '1', '2', num-elements-1, num-elements, and num-elements+1.

Any value greater than or equal to num-elements results the entire result set, same as when "limit" is unspecified.

These vector tests assume the target "/example-social:members/member=alice/favorites/uint8-numbers", which has six values, thus the edge condition "limit" values are: '1', '2', '5', '6', and '7'.

A.3.1.1. limit=1
REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    -
    Limit:     1
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [17],
  "@example-social:uint8-numbers": [
     {
        "ietf-list-pagination:remaining": 5
     }
   ]
}
~~~~
A.3.1.2. limit=2
REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    -
    Limit:     2
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [17, 13],
  "@example-social:uint8-numbers": [
     {
        "ietf-list-pagination:remaining": 4
     }
   ]
}
~~~~

#### A.3.1.3. limit=5

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    -
    Limit:     5
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [17, 13, 11, 7, 5],
  "@example-social:uint8-numbers": [
     {
        "ietf-list-pagination:remaining": 1
     }
   ]
}
~~~~
#### A.3.1.4. limit=6

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    -
    Limit:     6
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [17, 13, 11, 7, 5, 3]
}
~~~~
#### A.3.1.5. limit=7

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    -
    Limit:     7
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [17, 13, 11, 7, 5, 3]
}
~~~~





### A.3.2. The "offset" Parameter

Noting that "offset" must be an unsigned number less than or equal to the num-elements, the edge condition values are '0', '1', '2', num-elements-1, num-elements, and num-elements+1.

These vector tests again assume the target "/example-social:members/member=alice/favorites/uint8-numbers", which has six values, thus the edge condition "limit" values are: '0', '1', '2', '5', '6', and '7'.

#### A.3.2.1. offset=0

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    0
    Limit:     -
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [17, 13, 11, 7, 5, 3]
}
~~~~
#### A.3.2.2. offset=1

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    1
    Limit:     -
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [13, 11, 7, 5, 3]
}
~~~~

#### A.3.2.3. offset=2

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    2
    Limit:     -
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [11, 7, 5, 3]
}
~~~~
#### A.3.2.4. offset=5

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    5
    Limit:     -
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [3]
}
~~~~

#### A.3.2.5. offset=6

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    6
    Limit:     -
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": []
}
~~~~
#### A.3.2.6. offset=7

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    7
    Limit:     -
~~~~
RESPONSE
~~~~
error-type: application
error-tag: invalid-value
error-app-tag: ietf-list-pagination:offset-out-of-range
~~~~



### A.3.3. The "cursor" Parameter

Noting that "cursor" is an opaque encoded value represented by a string, which addresses an element in a list.

The default value is empty, which is the same as supplying the cursor value for the first element in the list.

These vector tests assume the target "/example-social:members/member" which has five members.

Note that response has added attributes describing the result set and position in pagination.

#### A.3.3.1. cursor=&limit=2
REQUEST
~~~~
Target: /example-social:members/member
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    -
    Limit:     2
    Cursor:    -
~~~~

RESPONSE

~~~~
{
  "example-social:member": [
    {
      "member-id": "bob",
      "email-address": "bob@example.com",
      "password": "$0$1543",
      "avatar": "BASE64VALUE=",
      "tagline": "Here and now, like never before.",
      "posts": {
        "post": [
          {
            "timestamp": "2020-08-14T03:32:25Z",
            "body": "Just got in."
          },
          {
            "timestamp": "2020-08-14T03:33:55Z",
            "body": "What's new?"
          },
          {
            "timestamp": "2020-08-14T03:34:30Z",
            "body": "I'm bored..."
          }
        ]
      },
      "favorites": {
        "decimal64-numbers": ["3.14159", "2.71828"]
      },
      "stats": {
        "joined": "2020-08-14T03:30:00Z",
        "membership-level": "standard",
        "last-activity": "2020-08-14T03:34:30Z"
      }
    },
    {
      "member-id": "eric",
      "email-address": "eric@example.com",
      "password": "$0$1543",
      "avatar": "BASE64VALUE=",
      "tagline": "Go to bed with dreams; wake up with a purpose.",
      "following": ["alice"],
      "posts": {
        "post": [
          {
            "timestamp": "2020-09-17T18:02:04Z",
            "title": "Son, brother, husband, father",
            "body": "What's your story?"
          }
        ]
      },
      "favorites": {
        "bits": ["two", "one", "zero"]
      },
      "stats": {
        "joined": "2020-09-17T19:38:32Z",
        "membership-level": "pro",
        "last-activity": "2020-09-17T18:02:04Z"
      }
    }
  ],
  "@example-social:member": [
    {
      "ietf-list-pagination:remaining": 3,
      "ietf-list-pagination:previous": "",
      "ietf-list-pagination:next": "YWxpY2U=" // alice
    }
  ]
}
~~~~

#### A.3.3.2. cursor="YWxpY2U="&limit=2
REQUEST
~~~~
Target: /example-social:members/member
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    -
    Limit:     2
    Cursor:    YWxpY2U=
~~~~
RESPONSE
~~~~
{
  "example-social:member": [
    {
      "member-id": "alice",
      "email-address": "alice@example.com",
      "password": "$0$1543",
      "avatar": "BASE64VALUE=",
      "tagline": "Every day is a new day",
      "privacy-settings": {
        "hide-network": false,
        "post-visibility": "public"
      },
      "following": ["bob", "eric", "lin"],
      "posts": {
        "post": [
          {
            "timestamp": "2020-07-08T13:12:45Z",
            "title": "My first post",
            "body": "Hiya all!"
          },
          {
            "timestamp": "2020-07-09T01:32:23Z",
            "title": "Sleepy...",
            "body": "Catch y'all tomorrow."
          }
        ]
      },
      "favorites": {
        "uint8-numbers": [17, 13, 11, 7, 5, 3],
        "int8-numbers": [-5, -3, -1, 1, 3, 5]
      },
      "stats": {
        "joined": "2020-07-08T12:38:32Z",
        "membership-level": "admin",
        "last-activity": "2021-04-01T02:51:11Z"
      }
    },
    {
      "member-id": "lin",
      "email-address": "lin@example.com",
      "password": "$0$1543",
      "privacy-settings": {
        "hide-network": true,
        "post-visibility": "followers-only"
      },
      "following": ["joe", "eric", "alice"],
      "stats": {
        "joined": "2020-07-09T12:38:32Z",
        "membership-level": "standard",
        "last-activity": "2021-04-01T02:51:11Z"
      }
    }
  ],
  "@example-social:member": [
    {
      "ietf-list-pagination:remaining": 1,
      "ietf-list-pagination:previous": "ZXJpYw==", // eric
      "ietf-list-pagination:next": "am9l" // joe
    }
  ]
}
~~~~
A.3.3.3. cursor="am9l"&limit=2
REQUEST
~~~~
Target: /example-social:members/member
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    -
    Limit:     2
    Cursor:    am9l
~~~~
RESPONSE
~~~~
{
  "example-social:member": [
    {
      "member-id": "joe",
      "email-address": "joe@example.com",
      "password": "$0$1543",
      "avatar": "BASE64VALUE=",
      "tagline": "Greatness is measured by courage and heart.",
      "privacy-settings": {
        "post-visibility": "unlisted"
      },
      "following": ["bob"],
      "posts": {
        "post": [
          {
            "timestamp": "2020-10-17T18:02:04Z",
            "body": "What's your status?"
          }
        ]
      },
      "stats": {
        "joined": "2020-10-08T12:38:32Z",
        "membership-level": "pro",
        "last-activity": "2021-04-01T02:51:11Z"
      }
    }
  ],
  "@example-social:member": [
    {
      "ietf-list-pagination:remaining": 0,
      "ietf-list-pagination:previous": "bGlu", // lin
      "ietf-list-pagination:next": ""
    }
  ]
}
~~~~
#### A.3.3.4. cursor="BASE64VALUE="
REQUEST

The cursor used does not exist in the datastore.
~~~~
Target: /example-social:members/member
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: -
    Offset:    -
    Limit:     -
    Cursor:    BASE64VALUE=
~~~~
RESPONSE
~~~~
error-type: application
error-tag: invalid-value
error-app-tag: ietf-list-pagination:cursor-not-found
~~~~



### A.3.4. The "direction" Parameter
Noting that "direction" is an enumeration with two values, the edge condition values are each defined enumeration.

The value "forwards" is sometimes known as the "default" value, as it produces the same result set as when "direction" is unspecified.

These vector tests again assume the target "/example-social:members/member=alice/favorites/uint8-numbers". The number of elements is relevant to the edge condition values.

It is notable that "uint8-numbers" is an "ordered-by" user leaf-list. Traversals are over the user-specified order, not the numerically-sorted order, which is what the "sort-by" parameter addresses. If this were an "ordered-by system" leaf-list, then the traversals would be over the system-specified order, again not a numerically-sorted order.

#### A.3.4.1. direction=forwards
REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: forwards
    Offset:    -
    Limit:     -
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [17, 13, 11, 7, 5, 3]
}
~~~~

#### A.3.4.2. direction=backwards
REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   -
    Direction: backwards
    Offset:    -
    Limit:     -
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [3, 5, 7, 11, 13, 17]
}
~~~~



### A.3.5. The "sort-by" Parameter
Noting that the "sort-by" parameter is a node identifier, there is not so much "edge conditions" as there are "interesting conditions". This section provides examples for some interesting conditions.

#### A.3.5.1. the target node's type
The section provides three examples, one for a "leaf-list" and two for a "list", with one using a direct descendent and the other using an indirect descendent.

##### A.3.5.1.1. type is a "leaf-list"
This example illustrates when the target node's type is a "leaf-list". Note that a single period (i.e., '.') is used to represent the nodes to be sorted.

This test again uses the target "/example-social:members/member=alice/favorites/uint8-numbers", which is a leaf-list.

REQUEST
~~~~
Target: /example-social:members/member=alice/favorites/uint8-numbers
  Pagination Parameters:
    Where:     -
    Sort-by:   .
    Direction: -
    Offset:    -
    Limit:     -
~~~~
RESPONSE
~~~~
{
  "example-social:uint8-numbers": [3, 5, 7, 11, 13, 17]
}
~~~~

##### A.3.5.1.2. type is a "list" and sort-by node is a direct descendent
This example illustrates when the target node's type is a "list" and a direct descendent is the "sort-by" node.

This vector test uses the target "/example-social:members/member", which is a "list", and the sort-by descendent node "member-id", which is the "key" for the list.

REQUEST
~~~~
Target: /example-social:members/member
  Pagination Parameters:
    Where:     -
    Sort-by:   member-id
    Direction: -
    Offset:    -
    Limit:     -
~~~~
RESPONSE

To make the example more understandable, an ellipse (i.e., "...") is used to represent a missing subtree of data.
~~~~
{
  "example-social:member": [
    {
      "member-id": "alice",
      ...
    },
    {
      "member-id": "bob",
      ...
    },
    {
      "member-id": "eric",
      ...
    },
    {
      "member-id": "joe",
      ...
    },
    {
      "member-id": "lin",
      ...
    }
  ]
}
~~~~

A.3.5.1.3. type is a "list" and sort-by node is an indirect descendent
This example illustrates when the target node's type is a "list" and an indirect descendent is the "sort-by" node.

This vector test uses the target "/example-social:members/member", which is a "list", and the sort-by descendent node "stats/joined", which is a "config false" descendent leaf. Due to "joined" being a "config false" node, this request would have to target the "member" node in the operational datastore.

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
