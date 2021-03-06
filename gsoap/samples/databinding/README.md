
XML Data Bindings                                                    {#mainpage}
=================

[TOC]

Introduction                                                            {#intro}
============

This is a detailed overview of the gSOAP XML data bindings concepts and
implementation.  At the end of this document two examples are given to
illustrate the application of data bindings.

The first simple example `address.cpp` shows how to use wsdl2h to bind an XML
schema to C++.  The C++ application reads and writes an XML file into and from
a C++ "address book" data structure.  The C++ data structure is an STL vector
of address objects.

The second example `graph.cpp` shows how XML is serialized as a tree, digraph,
and cyclic graph.  The digraph and cyclic graph serialization rules are similar
to SOAP 1.1/1.2 encoded multi-ref elements with id-ref attributes to link
elements through IDREF XML references, creating a an XML graph with pointers to
XML nodes.

These examples demonstrate XML data bindings only for relatively simple data
structures and types.  The gSOAP tools support more than just these type of
structures, which we will explain in the next sections.  Support for XML schema
components is practically unlimited.  The wsdl2h tool maps schemas to C and C++
using built-in intuitive mapping rules, while allowing the mappings to be
customized using a `typemap.dat` file with mapping instructions for wsdl2h.

The information in this document is applicable to gSOAP 2.8.26 and higher, which
supports C++11 features.  However, C++11 is not required to use this material
and follow the example, unless we need smart pointers and scoped enumerations.
While most of the examples in this document are given in C++, the concepts also
apply to C with the exception of containers, smart pointers, classes and their
methods.  None of these exceptions limit the use of the gSOAP tools for C in any
way.

The data binding concepts described in this document were first envisioned in
1999 by Prof. van Engelen at the Florida State University (the project was
named "stub/skeleton compiler").  The first articles on gSOAP appeared in 2002.
The principle of mapping XSD components to C/C++ types and vice versa is now
widely adopted in systems and programming languages, including Java web
services and by C# WCF.


Mapping WSDL and XML schemas to C/C++                                   {#tocpp}
=====================================

To convert WSDL and XML schemas (XSD files) to code, we first use the wsdl2h
command to generate the data binding interface code that is saved to a special
gSOAP header file:

    wsdl2h [options] -o file.h ... XSD and WSDL files ...

This command converts WSDL and XSD files to C++ (or pure C with wsdl2h option
`-c`) and saves a data binding interface file `file.h` that uses familar C/C++
syntax extended with `gsoap` directives and includes notational conventions
to declare C/C++ types and functions that are associated with these bindings.

The WSDL 1.1/2.0, SOAP 1.1/1.2, and XSD 1.0/1.1 standards are supported by the
gSOAP tools.  In addition, the most popular WS specifications are also
supported, including WS-Addressing, WS-ReliableMessaging, WS-Discovery,
WS-Security, WS-Policy, WS-SecurityPolicy, and WS-SecureConversation.

This document focusses on XML data bindings and mapping C/C++ to XML 1.0/1.1
and XSD 1.0/1.1.  This covers all of the following standard XSD components with
their optional `[ attributes ]` properties:

    schema         [targetNamespace, version, elementFormDefault,
                    attributeFormDefault, defaultAttributes]
    attribute      [name, ref, type, use, default, fixed, form,
                    targetNamespace, wsdl:arrayType]
    element        [name, ref, type, default, fixed, form, nillable, abstract,
                    substitutionGroup, minOccurs, maxOccurs, targetNamespace]
    simpleType     [name]
    complexType    [name, abstract, mixed, defaultAttributesApply]
    all
    choice         [minOccurs, maxOccurs]
    sequence       [minOccurs, maxOccurs]
    group          [name, ref, minOccurs, maxOccurs]
    attributeGroup [name, ref]
    any            [minOccurs, maxOccurs]
    anyAttribute

And also the following standard XSD components:

    import              imports a schema into the importing schema for referencing
    include             include schema component definitions into a schema
    override            override by replacing schema component definitions
    redefine            extend or restrict schema component definitions
    annotation          annotates a component

The XSD facets and their mappings to C/C++ are:

    enumeration         maps to enum
    simpleContent       maps to class/struct wrapper with __item member
    complexContent      maps to class/struct
    list                maps to enum* bitmask (enum* enumerates up to 64 bit masks)
    extension           through inheritance
    restriction         partly through inheritance and redeclaration
    length              restricts content length
    minLength           restricts content length
    maxLength           restricts content length
    minInclusive        restricts numerical value range
    maxInclusive        restricts numerical value range
    minExclusive        restricts numerical value range
    maxExclusive        restricts numerical value range
    precision           maps to float/double but constraint is not validated
    scale               maps to float/double but constraint is not validated
    totalDigits         maps to float/double but constraint is not validated
    fractionDigits      maps to float/double but constraint is not validated
    pattern             must define soap::fsvalidate callback to validate patterns
    union               maps to string of values

All primitive XSD types are supported, including but not limited to the
following XSD types:

    anyType             maps to _XML string with literal XML content (or DOM with wsdl2h option -d)
    anyURI              maps to string
    string              maps to string (char*/wchar_t*/std::string/std::wstring)
    boolean             maps to bool (C++) or enum xsd__boolean (C)
    byte                maps to char (int8_t)
    short               maps to short (int16_t)
    int                 maps to int (int32_t)
    long                maps to LONG64 (long long and int64_t)
    unsignedByte        maps to unsigned char (uint8_t)
    unsignedShort       maps to unsigned short (uint16_t)
    unsignedInt         maps to unsigned int (uint32_t)
    unsignedLong        maps to ULONG64 (unsigned long long and uint64_t)
    float               maps to float
    double              maps to double
    integer             maps to string or #import "custom/int128.h"
    decimal             maps to string or #import "custom/long_double.h"
    precisionDecimal    maps to string
    duration            maps to string or #import "custom/duration.h"
    dateTime            maps to time_t or #import "custom/struct_tm.h"
    time                maps to string or #import "custom/long_time.h"
    date                maps to string or #import "custom/struct_tm_date.h"
    hexBinary           maps to class/struct xsd__hexBinary
    base64Bianry        maps to class/struct xsd__base64Binary
    QName               maps to _QName (URI normalization rules are applied)

All other primitive XSD types not listed above are mapped to strings, by
generating a typedef.  For example, xsd:token is bound to a C++ or C string,
which associates a value space to the type with the appropriate XSD type name
used by the soapcpp2-generated serializers:

    typedef std::string  xsd__token;  // C++
    typedef char        *xsd__token;  // C (wsdl2h option -c)

It is possible to remap types by adding the appropriate mapping rules to
`typemap.dat` as we will explain in more detail in the next section.

Imported custom serializers are intended to extend the C/C++ type bindings when
the default binding to string is not satisfactory to your taste and if the
target platform supports these C/C++ types.  To add custom serializers to
`typemap.dat` for wsdl2h, see [Adding custom serializers](#custom) below.


Using typemap.dat to customize data bindings                          {#typemap}
============================================

We use a `typemap.dat` file to redefine namespace prefixes and to customize
type bindings for the the generated header files produced by the wsdl2h tool.
The `typemap.dat` is the default file processed by wsdl2h.  Use wsdl2h option
`-t` to specify a different file.

Declarations in `typemap.dat` can be broken up over multiple lines by
continuing on the next line by ending each line to be continued with a
backslash `\`.  The limit is 4095 characters per line, whether the line is
broken up or not.


XML namespace bindings                                               {#typemap1}
----------------------

The wsdl2h tool generates C/C++ type declarations that use `ns1`, `ns2`, etc.
as schema-binding URI prefixes.  These default prefixes are generated somewhat
arbitrarily for each schema targetNamespace URI, meaning that their ordering
may change depending on the WSDL and XSD order of processing with wsdl2h.

Therefore, it is **strongly recommended** to declare your own prefix for each
schema URI in `typemap.dat` to reduce maintaince effort of your code.  This
is more robust when anticipating possible changes of the schema(s) and/or the
binding URI(s) and/or the tooling algorithms.

The first and foremost important thing to do is to define prefix-URI bindings
for our C/C++ code by adding the following line(s) to our `typemap.dat` or make
a copy of this file and add the line(s) that bind our choice of prefix name to
each URI:

    prefix = "URI"

For example:

    g = "urn:graph"

This produces `g__name` C/C++ type names that are bound to the "urn:graph"
schema by association of `g` to the generated C/C++ types.

This means that `<g:name xmlns:g="urn:graph">` is parsed as an instance of a
`g__name` C/C++ type.  Also `<x:name xmlns:x="urn:graph">` parses as an instance
of `g__name`, because the prefix `x` has the same URI value `urn:graph`.
Prefixes in XML have local scopes (like variables in a block).

The first run of wsdl2h will reveal the URIs, so you do not need to search
WSDLs and XSD files for all of the target namespaces.  Just copy them from the
generated header file after the first run into `typemap.dat` for editing.


XSD type bindings                                                    {#typemap2}
-----------------

Custom C/C++ type bindings can be declared in `typemap.dat` to associate C/C++
types with specific schema types.  These type bindings have four parts:

    prefix__type = declaration | use | ptruse

where

- `prefix__type` is the schema type to be customized (the `prefix__type` name
  uses the common double underscore naming convention);
- `declaration` declares the C/C++ type in the wsdl2h-generated header file.
  This part can be empty if no explicit declaration is needed;
- `use` is an optional part that specifies how the C/C++ type is used in the
  code.  When omitted, it is the same as `prefix__type`;
- `ptruse` is an optional part that specifies how the type is used as a
  pointer type.  By default it is the `use` type name with a `*` or C++11
  `std::shared_ptr<>` when enabled (see further below).

For example, to map xsd:duration to a `long long` (`LONG64`) type that holds
millisecond duration values, we can use the custom serializer declared in
`custom/duration.h` by adding the following line to `typemap.dat`:

    xsd__duration = #import "custom/duration.h"

Here, we omitted the second field, because `xsd__duration` is the name that
wsdl2h uses to identify and use this type for our code.  The third field is
omitted to let wsdl2h use `xsd__duration *` for pointers or
`std::shared_ptr<xsd__duration>` if smart pointers are enabled.

To map xsd:string to `wchar_t*` wide strings:

    xsd__string = | wchar_t* | wchar_t*

Note that the first field is empty, because `wchar_t` is a C type and does not
need to be declared.  A `ptruse` field is given so that we do not end up
generating the wrong pointer types, such as `wchar_t**` and
`std::shared_ptr<wchar_t>`.

When the auto-generated declaration should be preserved but the `use` or
`ptruse` fields replaced, then we use an ellipsis for the declaration part:

    prefix__type = ... | use | ptruse

This is useful to map schema polymorphic types to C types for example, where we
need to be able to both handle a base type and its extensions as per schema
extensibility.  Say we have a base type called ns:base that is extended, then we
can remap this to a C type that permits referening the extended types via a
`void*` as follows:

    ns__base = ... | int __type_base; void*

such that `__type_base` and `void*` are used to (de)serialize any data type,
including base and its derived types.


Custom serializers for XSD types                                       {#custom}
--------------------------------

In the previous part we saw how a custom serializer is used to bind
xsd:duration to a `long long` (`LONG64` or `int64_t`) type to store millisecond
duration values:

    xsd__duration = #import "custom/duration.h"

The `xsd__duration` type is an alias of `long long` (`LONG64` or `int64_t`).

While wsdl2h will use this binding declared in `typemap.dat` automatically, you
will also need to compile `custom/duration.c`.  Each custom serializer has a
header file and an implementation file written in C.  You can compile these in
C++ (rename files to `.cpp` if needed).

We will discuss the custom serializers that are available to you.

### xsd:integer                                                      {#custom-1}

The wsdl2h tool maps xsd:integer to a string by default.  To map xsd:integer to
the 128 bit big int type `__int128_t`:

    xsd__integer = #import "custom/int128.h"

The `xsd__integer` type is an alias of `__int128_t`.

@warning Beware that the xsd:integer value space of integers is in principle
unbounded and values can be of arbitrary length.  A value range fault
`SOAP_TYPE` (value exceeds native representation) or `SOAP_LENGTH` (value
exceeds range bounds) will be thrown by the deserializer if the value is out of
range.

Other XSD integer types that are restrictions of xsd:integer, are
xsd:nonNegativeInteger and xsd:nonPositiveInteger, which are further restricted
by xsd:positiveInteger and xsd:negativeInteger.  To bind these types to
`__int128_t` we should also add the following definitions to `typemap.dat`:

    xsd__nonNegativeInteger = typedef xsd__integer xsd__nonNegativeInteger 0 :    ;
    xsd__nonPositiveInteger = typedef xsd__integer xsd__nonPositiveInteger   : 0  ;
    xsd__positiveInteger    = typedef xsd__integer xsd__positiveInteger    1 :    ;
    xsd__negativeInteger    = typedef xsd__integer xsd__negativeInteger      : -1 ;

@note If `__int128_t` 128 bit integers are not supported on your platform and if it
is certain that xsd:integer values are within 64 bit value bounds for your
application's use, then you can map this type to `LONG64`:

    xsd__integer = typedef LONG64 xsd__integer;

@note Again, a value range fault `SOAP_TYPE` or `SOAP_LENGTH` will be thrown by
the deserializer if the value is out of range.

@see Section [Numerical types](#toxsd5).

### xsd:decimal                                                      {#custom-2}

The wsdl2h tool maps xsd:decimal to a string by default.  To map xsd:decimal to
extended precision floating point:

    xsd__decimal = #import "custom/long_double.h" | long double

By contrast to all other custom serializers, this serializer enables `long
double` natively without requiring a new binding name (`xsd__decimal` is NOT
defined).

If your system supports `<quadmath.h>` quadruple precision floating point
`__float128`, you can map xsd:decimal to `xsd__decimal` that is an alias of
`__float128`:

    xsd__decimal = #import "custom/float128.h"

@warning Beware that xsd:decimal is in principle a decimal value with arbitraty
lengths.  A value range fault `SOAP_TYPE` will be thrown by the deserializer if
the value is out of range.

In the XML payload the special values `INF`, `-INF`, `NaN` represent plus or
minus infinity and not-a-number, respectively.

@see Section [Numerical types](#toxsd5).

### xsd:dateTime                                                     {#custom-3}

The wsdl2h tool maps xsd:dateTime to `time_t` by default.

The trouble with `time_t` is that it is limited to dates between 1970 and 2038
(until its decided by 2038 to widen the bit representation of `time_t`).

For this reason `struct tm` should be used to represent wider date ranges.  This
custom serializer avoids using date and time information in `time_t`.  You get
the raw date and time information.  You only lose the day of the week
information.  It is always Sunday (`tm_wday=0`).

To map xsd:dateTime to `xsd__dateTime` which is an alias of `struct tm`:

    xsd__dateTime = #import "custom/struct_tm.h"

If the limited date range of `time_t` is not a problem but you want to increase
the time precision with fractional seconds, then we suggest to map xsd:dateTime
to `struct timeval`:

    xsd__dateTime = #import "custom/struct_timeval.h"

If the limited date range of `time_t` is not a problem but you want to use the
C++11 time point type `std::chrono::system_clock::time_point` (which internally
uses `time_t`):

    xsd__dateTime = #import "custom/chrono_time_point.h"

Again, we should make sure that the dates will not exceed the date range when
using the default `time_t` binding for xsd:dateTime or when binding
xsd:dateTime to `struct timeval` or to `std::chrono::system_clock::time_point`.
These are safe to use in applications that use xsd:dateTime to record date
stamps within a given window.  Otherwise, we recommend the `struct tm` custom
serializer.  You could even map xsd:dateTime to a plain string (use `char*` with
C and `std::string` with C++).  For example:

    xsd__dateTime = | char*

@see Section [Date and time types](#toxsd7).

### xsd:date                                                         {#custom-4}

The wsdl2h tool maps xsd:date to a string by default.  We can map xsd:date to
`struct tm`:

    xsd__date = #import "custom/struct_tm_date.h"

The `xsd__date` type is an alias of `struct tm`.  The serializer ignores the
time part and the deserializer only populates the date part of the struct,
setting the time to 00:00:00.  There is no unreasonable limit on the date range
because the year field is stored as an integer (`int`).

@see Section [Date and time types](#toxsd7).

### xsd:time                                                         {#custom-5}

The wsdl2h tool maps xsd:time to a string by default.  We can map xsd:time to
an `unsigned long long` (`ULONG64` or `uint64_t`) integer with microsecond time
precision:

    xsd__time = #import "custom/long_time.h"

This type represents 00:00:00.000000 to 23:59:59.999999, from `0` to an upper
bound of `86399999999`.  A microsecond resolution means that a 1 second
increment requires an increment of 1000000 in the integer value.  The serializer
adds a UTC time zone.

@see Section [Date and time types](#toxsd7).

### xsd:duration                                                     {#custom-6}

The wsdl2h tool maps xsd:duration to a string by default, unless xsd:duration
is mapped to a `long long` (`LONG64` or `int64_t`) type with with millisecond
(ms) time duration precision:

    xsd__duration = #import "custom/duration.h"

The `xsd__duration` type is a 64 bit signed integer that can represent
106,751,991,167 days forwards (positive) and backwards (negative) in time in
increments of 1 ms (1/1000 of a second).

Rescaling of the duration value by may be needed when adding the duration value
to a `time_t` value, because `time_t` may or may not have a seconds resolution,
depending on the platform and possible changes to `time_t`.

Rescaling is done automatically when you add a C++11 `std::chrono::nanoseconds`
value to a `std::chrono::system_clock::time_point` value.  To use
`std::chrono::nanoseconds` as xsd:duration:

    xsd__duration = #import "custom/chrono_duration.h"

This type can represent 384,307,168 days (2^63 nanoseconds) forwards and
backwards in time in increments of 1 ns (1/1,000,000,000 of a second).

Certain observations with respect to receiving durations in years and months
apply to both of these serializer decoders for xsd:duration.

@see Section [Time duration types](#toxsd8).


Class/struct member additions                                        {#typemap3}
-----------------------------

All generated classes and structs can be augmented with additional
members such as methods, constructors and destructors, and private members:

    prefix__type = $ member-declaration

For example, we can add method declarations and private members to a class, say
`ns__record` as follows:

    ns__record = $ ns__record(const ns__record &);  // copy constructor
    ns__record = $ void print();                    // a print method
    ns__record = $ private: int status;             // a private member

Note that method declarations cannot include any code, because soapcpp2's input
permits only type declarations, not code.


Replacing XSD types by equivalent alternatives                       {#typemap4}
----------------------------------------------

Type replacements can be given to replace one type entirely with another given
type:

    prefix__type1 == prefix__type2

This replaces all `prefix__type1` by `prefix__type2` in the wsdl2h output.

@warning Do not agressively replace types, because this can cause XML
validation to fail when a value-type mismatch is encountered in the XML input.
Therefore, only replace similar types with other similar types that are wider
(e.g.  `short` by `int` and `float` by `double`).


The built-in typemap.dat variables $CONTAINER and $POINTER           {#typemap5}
----------------------------------------------------------

The `typemap.dat` `$CONTAINER` variable defines the container to emit in the
generated declarations, which is `std::vector` by default.  For example, to emit
`std::list` as the container in the wsdl2h-generated declarations:

    $CONTAINER = std::list

The `typemap.dat` `$POINTER` variable defines the smart pointer to emit in the
generated declarations, which replaces the use of `*` pointers.  For example:

    $POINTER = std::shared_ptr

Not all pointers in the generated output can be replaced by smart pointers.
Regular pointers are still used as union members and for pointers to arrays of
objects.

@note The standard smart pointer `std::shared_ptr` is generally safe to use.
Other smart pointers such as `std::unique_ptr` and `std::auto_ptr` may cause
compile-time errors when classes have smart pointer members but no copy
constructor (a default copy constructor).  A copy constructor is required for
non-shared smart pointer copying or swapping.

Alternatives to `std::shared_ptr` of the form `NAMESPACE::shared_ptr` can be
assigned to `$POINTER` when the namespace `NAMESPACE` also implements
`NAMESPACE::make_shared` and when the shared pointer class provides `reset()`
and`get()` methods and the dereference operator.  For example Boost
`boost::shared_ptr`:

    [
    #include <boost/shared_ptr.hpp>
    ]
    $POINTER = boost::shared_ptr

The user-defined content between `[` and `]` ensures that we include the Boost
header files that are needed to support `boost::shared_ptr` and
`boost::make_shared`.


User-defined content                                                 {#typemap6}
--------------------

Any other content to be generated by wsdl2h can be included in `typemap.dat` by
enclosing it within brackets `[` and `]` anywhere in the `typemap.dat` file.
Each of the two brackets MUST appear at the start of a new line.

For example, we can add an `#import "wsa5.h"` directive to the wsdl2h-generated
output as follows:

    [
    #import "import/wsa5.h"
    ]

which emits the `#import "import/wsa5.h"` literally at the start of the
wsdl2h-generated header file.


Mapping C/C++ to XML schema                                             {#toxsd}
===========================

The soapcpp2 command generates the data binding implementation code from a data
binding interface `file.h`:

    soapcpp2 [options] file.h

where `file.h` is a gSOAP header file that declares the XML data binding
interface.  The `file.h` is typically generated by wsdl2h, but we can also
declare one ourself.  If so, we add gSOAP directives and declare in this file
all our C/C++ types we want to serialize in XML.  We can also declare functions
that will be converted to service operations by soapcpp2.

Global function declarations define service operations, which are of the form:

    int ns__name(arg1, arg2, ..., argn, result);

where `arg1`, `arg2`, ..., `argn` are formal argument declarations of the input
and `result` is a formal argument for the output, which must be a pointer or
reference to the result object to be populated.  More information can be found
in the gSOAP user guide.


Overview of serializable C/C++ types                                   {#toxsd1}
------------------------------------

The following C/C++ types are supported by soapcpp2 and mapped to XSD types
and constructs.  See the subsections below for more details or follow the links.

### List of Boolean types

    bool                      C++ bool
    enum xsd__boolean         C alternative bool
 
@see Section [C++ bool and C alternative](#toxsd3).

### List of enumeration and bitmask types

    enum                      enumeration
    enum class                C++11 scoped enumeration (soapcpp2 -c++11)
    enum*                     a bitmask that enumerates values 1, 2, 4, 8, ...
    enum* class               C++11 scoped enumeration bitmask (soapcpp2 -c++11)

@see Section [Enumerations and bitmasks](#toxsd4).

### List of numerical types

    char                      byte
    short                     16 bit integer
    int                       32 bit integer
    long                      32 bit integer
    LONG64                    64 bit integer
    xsd__integer              128 bit __int128_t integer, use #import "custom/int128.h"
    long long                 same as LONG64
    unsigned char             unsigned byte
    unsigned short            unsigned 16 bit integer
    unsigned int              unsigned 32 bit integer
    unsigned long             unsigned 32 bit integer
    ULONG64                   unsigned 64 bit integer
    unsigned long long        same as ULONG64
    int8_t                    same as char
    int16_t                   same as short
    int32_t                   same as int
    int64_t                   same as LONG64
    uint8_t                   same as unsigned char
    uint16_t                  same as unsigned short
    uint32_t                  same as unsigned int
    uint64_t                  same as ULONG64
    size_t                    transient type (not serializable)
    float                     32 bit float
    double                    64 bit float
    long double               extended precision float, use #import "custom/long_double.h"
    xsd__decimal              <quadmath.h> 128 bit __float128 quadruple precision float, use #import "custom/float128.h"
    typedef                   declares a type name, with optional value range and string length bounds

@see Section [Numerical types](#toxsd5).

### List of string types.

    char*                     string
    wchar_t*                  wide string
    std::string               C++ string
    std::wstring              C++ wide string
    char[N]                   fixed-size string, requires soapcpp2 option -b
    _QName                    normalized QName content
    _XML                      literal XML string content
    typedef                   declares a type name, may restrict string length

@see Section [String types](#toxsd6).

### List of date and time types

    time_t                    date and time point since epoch
    struct tm                 date and time point, use #import "custom/struct_tm.h"
    struct tm                 date point, use #import "custom/struct_tm_date.h"
    struct timeval            date and time point, use #import "custom/struct_timeval.h"
    unsigned long long        time point in microseconds, use #import "custom/long_time.h"
    std::chrono::system_clock::time_point
                              date and time point, use #import "custom/chrono_time_point.h"

@see Section [Date and time types](#toxsd7).

### List of time duration types

    long long                 duration in milliseconds, use #import "custom/duration.h"
    std::chrono::nanoseconds  duration in nanoseconds, use #import "custom/chrono_duration.h"

@see Section [Time duration types](#toxsd8).

### List of classes and structs

    class                     C++ class with single inheritance only
    struct                    C struct or C++ struct without inheritance
    T*                        pointer to type T
    T[N]                      fixed-size array of type T
    std::shared_ptr<T>        C++11 smart shared pointer
    std::unique_ptr<T>        C++11 smart pointer
    std::auto_ptr<T>          C++ smart pointer
    std::deque<T>             use #import "import/stldeque.h"
    std::list<T>              use #import "import/stllist.h"
    std::vector<T>            use #import "import/stlvector.h"
    std::set<T>               use #import "import/stlset.h"
    template<T> class         a container with begin(), end(), size(), clear(), and insert() methods
    union                     requires a discriminant member
    void*                     requires a __type member to indicate the type of object pointed to

@see Section [Classes and structs](#toxsd9).

### List of special classes and structs

    Array                     single and multidimensional SOAP Arrays
    xsd__hexBinary            binary content
    xsd__base64Binary         binary content and optional MIME/MTOM attachments
    Wrapper                   complexTypes with simpleContent

@see Section [Special classes and structs](#toxsd10).


Colon notation versus name prefixing                                   {#toxsd2}
------------------------------------

To bind C/C++ type names to XSD types, a simple form of name prefixing is used
by the gSOAP tools by prepending the XML namespace prefix to the C/C++ type
name with a pair of undescrores.  This also ensures that name clashes cannot
occur when multiple WSDL and XSD files are converted to C/C++.  Also, C++
namespaces are not sufficiently rich to capture XML schema namespaces
accurately, for example when class members are associated with schema elements
defined in another XML namespace and thus the XML namespace scope of the
member's name is relevant, not just its type.

However, from a C/C++ centric point of view this can be cumbersome.  Therefore,
colon notation is an alternative to physically augmenting C/C++ names with
prefixes.

For example, the following class uses colon notation to bind the `record` class
to the `urn:types` schema:

    //gsoap ns schema namespace: urn:types
    class ns:record        // binding 'ns:' to a type name
    {
     public:
      std::string name;
      uint64_t    SSN;
      ns:record   *spouse;  // using 'ns:' with the type name
      ns:record();          // using 'ns:' here too
      ~ns:record();         // and here
    };

The colon notation is stripped away by soapcpp2 when generating the data
binding implementation code for our project.  So the final code just uses
`record` to identify this class and its constructor/destructor.

When using colon notation we have to be consistent and not use colon notation
mixed with prefixed forms.  The name `ns:record` differs from `ns__record`,
because `ns:record` is compiled to an unqualified `record` name.


C++ Bool and C alternative                                             {#toxsd3}
--------------------------

The C++ `bool` type is bound to built-in XSD type xsd:boolean.

The C alternative is to define an enumeration:

    enum xsd__boolean { false_, true_ };

or by defining an enumeration in C with pseudo-scoped enumeration constants:

    enum xsd__boolean { xsd__boolean__false, xsd__boolean__true };

The XML value space of these types is `false` and `true`, but also accepts `0`
and `1` as values.

To prevent name clashes, `false_` and `true_` have an underscore which are
removed in the XML value space.


Enumerations and bitmasks                                              {#toxsd4}
-------------------------

Enumerations are mapped to XSD simpleType enumeration restrictions of
xsd:string, xsd:QName, and xsd:long.

Consider for example:

    enum ns__Color { RED, WHITE, BLUE };

which maps to a simpleType restriction of xsd:string in the soapcpp2-generated
schema:

    <simpleType name="Color">
      <restriction base="xsd:string">
        <enumeration value="RED"/>
        <enumeration value="WHITE"/>
        <enumeration value="BLUE"/>
      </restriction>
    </simpleType>

Enumeration name constants can be pseudo-scoped to prevent name clashes,
because enumeration name constants have a global scope in C and C++:

    enum ns__Color { ns__Color__RED, ns__Color__WHITE, ns__Color__BLUE };

We can also use C++11 scoped enumerations to prevent name clashes:

    enum class ns__Color : int { RED, WHITE, BLUE };

Here, the enumeration class base type `: int` is optional.  In place of `int`
in the example above, we can also use `int8_t`, `int16_t`, `int32_t`, or
`int64_t`.

The XML value space of the enumertions defined above is `RED`, `WHITE`, and
`BLUE`.

Prefix-qualified enumeration name constants are mapped to simpleType
restrictions of xsd:QName, for example:

    enum ns__types { xsd__int, xsd__float };

which maps to a simpleType restriction of xsd:QName in the soapcpp2-generated
schema:

    <simpleType name="types">
      <restriction base="xsd:QName">
        <enumeration value="xsd:int"/>
        <enumeration value="xsd:float"/>
      </restriction>
    </simpleType>

Enumeration name constants can be pseudo-numeric as follows:

    enum ns__Primes { _3 = 3, _5 = 5, _7 = 7, _11 = 11 };

which maps to a simpleType restriction of `xsd:long`:

    <simpleType name="Color">
      <restriction base="xsd:long">
        <enumeration value="3"/>
        <enumeration value="5"/>
        <enumeration value="7"/>
        <enumeration value="11"/>
      </restriction>
    </simpleType>

The XML value space of this type is `3`, `5`, `7`, and `11`.

Besides (pseudo-) scoped enumerations, another way to prevent name clashes
accross enumerations is to start an enumeration name constant with one
underscore or followed it by any number of underscores, which makes it
unique.  The leading and trailing underscores are removed in the XML value
space.

    enum ns__ABC { A, B, C };
    enum ns__BA  { B, A };      // BAD: B = 1 but B is already defined as 2
    enum ns__BA_ { B_, A_ };    // OK

The gSOAP soapcpp2 tool permits reusing enumeration name constants across
(non-scoped) enumerations as long as these values are assigned the same
constant.  Therefore, the following is permitted:

    enum ns__Primes { _3 = 3, _5 = 5, _7 = 7, _11 = 11 };
    enum ns__Throws { _1 = 1, _2 = 2, _3 = 3, _4 = 4, _5 = 5, _6 = 6 };

A bitmask type is an `enum*` "product" enumeration with a geometric,
power-of-two sequence of values assigned to the enumeration constants:

    enum* ns__Options { SSL3, TLS10, TLS11, TLS12 };

where the product enum assigns 1 to `SSL3`, 2 to `TLS10`, 4 to `TLS11`, and 8
to `TLS12`, which allows these enumeration constants to be used in composing
bitmasks with `|` (bitwise or) `&` (bitwise and), and `~` (bitwise not):

    enum ns__Options options = (enum ns__Options)(SSL3 | TLS10 | TLS11 | TLS12);
    if (options & SSL3)  // if SSL3 is an option, warn and remove from options
    {
      warning();
      options &= ~SSL3;
    }

The bitmask type maps to a simpleType list restriction of xsd:string in the
soapcpp2-generated schema:

    <simpleType name="Options">
      <list>
        <restriction base="xsd:string">
          <enumeration value="SSL3"/>
          <enumeration value="TLS10"/>
          <enumeration value="TLS11"/>
          <enumeration value="TLS12"/>
        </restriction>
      </list>
    </simpleType>

The XML value space of this type consists of all 16 possible subsets of the
four values, represented by an XML string with space-separated values.  For
example, the bitmask `TLS10 | TLS11 | TLS12` equals 14 and is represented in by
the XML string `TLS10 TLS11 TLS12`.

We can also use C++11 scoped enumerations with bitmasks:

    enum* class ns__Options { SSL3, TLS10, TLS11, TLS12 };

The base type of a scoped enumeration bitmask, when explicitly given, is
ignored.  The base type is either `int` or `int64_t`, depending on the number
of constants enumerated in the bitmask.

To convert `enum` name constants and bitmasks to a string, we use the
auto-generated function for enum `T`:

    const char *soap_T2s(struct soap*, enum T val)

To convert a string to an `enum` constant or bitmask, we use the auto-generated
function

    int soap_s2T(struct soap*, const char *str, enum T *val)
  
This function takes the name (or names, space-separated for bitmasks) of
the enumeration constant in a string `str`.  Names should be given without the
pseudo-scope prefix and without trailing underscores.  The function sets `val`
to the corresponding integer enum constant or to a bitmask.  The function
returns `SOAP_OK` (zero) on success or an error if the string is not a valid
enumeration name.


Numerical types                                                        {#toxsd5}
---------------

Integer and floating point types are mapped to the equivalent built-in XSD
types with the same sign and bit width.

The `size_t` type is transient (not serializable) because its width is platform
dependent.  We recommend to use `uint64_t` instead.

The XML value space of integer types are their decimal representations without
loss of precision.

The XML value space of floating point types are their decimal representations.
The decimal representations are formatted with the printf format string "%.9G"
for floats and the printf format string "%.17lG" for double.  To change the
format strings, we can assign new strings to the following `struct soap`
context members:

    soap.float_format       = "%g";
    soap.double_format      = "%lg";
    soap.long_double_format = "%Lg";

Note that decimal representations may result in a loss of precision of the
least significant decimal.  Therefore, the format strings that are used by
default are sufficiently precise to avoid loss, but this may result in long
decimal fractions in the XML value space.

The `long double` extended floating point type requires a custom serializer:

    #import "custom/long_double.h"
    ... use long double ...

You can now use `long double`, which has a serializer that serializes this type
as `xsd:decimal`.  Compile and link your code with `custom/long_double.c`.

The value space of floating point values includes the special values `INF`,
`-INF`, and `NaN`.  You can check a value for plus or minus infinity and
not-a-number as follows:

    soap_isinf(x) && x > 0  // is x INF?
    soap_isinf(x) && x < 0  // is x -INF?
    soap_isnan(x)           // is x NaN?

To assign these values, use:

    // x is float       // x is double, long double, or __float128
    x = FLT_PINFY;      x = DBL_PINFTY;
    x = FLT_NINFY;      x = DBL_NINFTY;
    x = FLT_NAN;        x = DBL_NAN;

If your system supports `__float128` then you can also use this 128 bit
floating point type with a custom serializer:

    #import "custom/float128.h"
    ... use xsd__decimal ...

Then use the `xsd__decimal` alias of `__float128`, which has a serializer.  Do
not use `__float128` directly, which is transient (not serializable).

To check for `INF`, `-INF`, and `NaN` of a `__float128` value use:

    isinfq(x) && x > 0  // is x INF?
    isinfq(x) && x < 0  // is x -INF?
    isnanq(x)           // is x NaN?

The range of a typedef-defined numerical type can be restricted using the range
`:` operator with inclusive lower and upper bounds.  For example:

    typedef int ns__narrow -10 : 10;

This maps to a simpleType restriction of xsd:int in the soapcpp2-generated
schema:

    <simpleType name="narrow">
      <restriction base="xsd:int">
        <minInclusive value="-10"/>
        <maxInclusive value="10"/>
      </restriction>
    </simpleType>

The lower and upper bound of a range are optional.  When omitted, values are
not bound from below or from above, respectively.

The range of a floating point typedef-defined type can be restricted within
floating point constant bounds.

Also with a floating point typedef a printf format pattern can be given of the
form `"%[width][.precision]f"` to format decimal values using the given width
and precision fields:

    typedef float ns__PH "%5.2f" 0.0 : 14.0;

This maps to a simpleType restriction of xsd:float in the soapcpp2-generated
schema:

    <simpleType name="PH">
      <restriction base="xsd:float">
        <totalDigits value="5"/>
        <fractionDigits value="2"/>
        <minInclusive value="0"/>
        <maxInclusive value="14"/>
      </restriction>
    </simpleType>

For exclusive bounds, we use the `<` operator instead of the `:` range
operator:

    typedef float ns__epsilon 0.0 < 1.0;

Values `eps` of `ns__epsilon` are restricted between `0.0 < eps < 1.0`.

This maps to a simpleType restriction of xsd:float in the soapcpp2-generated
schema:

    <simpleType name="epsilon">
      <restriction base="xsd:float">
        <minExclusive value="0"/>
        <maxExclusive value="1"/>
      </restriction>
    </simpleType>

To make just one of the bounds exclusive, while keeping the other bound
inclusive, we add a `<` on the left or on the right side of the range ':'
operator.  For example:

    typedef float ns__pos 0.0 < : ;  // 0.0 < pos
    typedef float ns__neg : < 0.0 ;  // neg < 0.0

It is valid to make both left and right side exclusive with `< : <` which is in
fact identical to the exlusive range `<` operator:

    typedef float ns__epsilon 0.0 < : < 1.0;  // 0.0 < eps < 1.0

It helps to think of the `:` as a placeholder of the value between the two
bounds, which is easier to memorize than the shorthand forms of bounds from
which the `:` is removed:

| Bounds     | Validation check | Shorthand |
| ---------- | ---------------- | --------- |
| 1 :        | 1 <= x           | 1         |
| 1 : 10     | 1 <= x <= 10     |           |
|   : 10     |      x <= 10     |           |
| 1 < : < 10 | 1 <  x <  10     | 1 < 10    |
| 1   : < 10 | 1 <= x <  10     |           |
|     : < 10 |      x <  10     |   < 10    |
| 1 < :      | 1 <  x           | 1 <       |
| 1 < : 10   | 1 <  x <= 10     |           |

Besides `float`, also `double` and `long double` values can be restricted.  For
example, consider a nonzero probability extended floating point precision type:

    #import "custom/long_double.h"
    typedef long double ns__probability "%16Lg" 0.0 < : 1.0;

Value range restrictions are validated by the parser for all inbound XML data.
A type fault `SOAP_TYPE` will be thrown by the deserializer if the value is out
of range.

Finally, if your system supports `__int128_t` then you can also use this 128
bit integer type with a custom serializer:

    #import "custom/int128.h"
    ... use xsd__integer ...

We use the `xsd__integer` alias of `__int128_t`, which has a serializer.  Do not
use `__int128_t` directly, which is transient (not serializable).

To convert numeric values to a string, we use the auto-generated function for
numeric type `T`:

    const char *soap_T2s(struct soap*, T val)

For numeric types `T`, the string returned is stored in an internal buffer, so
you MUST copy it to keep it from being overwritten.  For example, use
`soap_strdup(struct soap*, const char*)`.

To convert a string to a numeric value, we use the auto-generated function

    int soap_s2T(struct soap*, const char *str, T *val)
  
where `T` is for example `int`, `LONG64`, `float`, `decimal` (the custom
serializer name of `long double`) or `xsd__integer` (the custom serializer name
of `__int128_t`).  The function `soap_s2T` returns `SOAP_OK` on success or an
error when the value is not numeric.  For floating point types, "INF", "-INF"
and "NaN" are valid strings to convert to numbers.


String types                                                           {#toxsd6}
------------

String types are mapped to the built-in xsd:string and xsd:QName XSD types.

The wide strings `wchar_t*` and `std::wstring` may contain Unicode that is
preserved in the XML value space.

Strings `char*` and `std::string` can only contain extended Latin, but we can
store UTF-8 content that is preserved in the XML value space when the `struct
soap` context is initialized with the flag `XML_C_UTFSTRING`.

@warning Beware that many XML 1.0 parsers reject all control characters (those
between `#x1` and `#x1F`) except `#x9`, `#xA`, and `#xD`.  With the newer XML
1.1 version parsers (including gSOAP) you should be fine.

The length of a string of a typedef-defined string type can be restricted:

    typedef std::string ns__password 6 : 16;

which maps to a simpleType restriction of xsd:string in the soapcpp2-generated
schema:

    <simpleType name="password">
      <restriction base="xsd:string">
        <minLength value="6"/>
        <maxLength value="16"/>
      </restriction>
    </simpleType>

String length restrictions are validated by the parser for inbound XML data.
A value length fault `SOAP_LENGTH` will be thrown by the deserializer if the
string is too long or too short.

In addition, an XSD regex pattern restriction can be associated with a string
typedef:

    typedef std::string ns__password "([a-zA-Z]|[0-9]|-)+" 6 : 16;

which maps to a simpleType restriction of xsd:string in the soapcpp2-generated
schema:

    <simpleType name="password">
      <restriction base="xsd:string">
        <pattern value="([a-zA-Z0-9]|-)+"/>
        <minLength value="6"/>
        <maxLength value="16"/>
      </restriction>
    </simpleType>

Pattern restrictions are validated by the parser for inbound XML data only if
the `soap::fsvalidate` and `soap::fwvalidate` callbacks are defined, see the
gSOAP user guide for more details.

Exclusive length bounds can be used with strings:

    typedef std::string ns__string255 : < 256;  // same as 0 : 255

Fixed-size strings (`char[N]`) are rare occurrences in the wild, but apparently
still used in some projects to store strings.  To facilitate fixed-size string
serialization, use soapcpp2 option `-b`.  For example:

    typedef char ns__buffer[10];  // requires soapcpp2 option -b

which maps to a simpleType restriction of xsd:string in the soapcpp2-generated
schema:

    <simpleType name="buffer">
      <restriction base="xsd:string">
        <maxLength value="9"/>
      </restriction>
    </simpleType>

Note that fixed-size strings MUST contain NUL-terminated text and SHOULD NOT
contain raw binary data.  Also, the length limitation is more restrictive for
UTF-8 content (enabled with the `SOAP_C_UTFSTRING`) that requires multibyte
character encodings.  As a consequence, UTF-8 content may be truncated to fit.

Note that raw binary data can be stored in a `xsd__base64Binary` or
`xsd__hexBinary` structure, or transmitted as a MIME attachment.

The built-in `_QName` type is a regular C string type (`char*`) that maps to
xsd:QName but has the added advantage that it holds normalized qualified names.
There are actually two forms of normalized QName content, to ensure any QName
is represented accurately and uniquely:

    prefix:name
    "URI":name

where the first form is used when the prefix (and the binding URI) is defined
in the namespace table and is bound to a URI (see the .nsmap file).  The second
form is used when the URI is not defined in the namespace table and therefore
no prefix is available to bind and normalize the URI to.

A `_QName` string may contain a sequence of space-separated QName values, not
just one, and all QName values are normalized to the format shown above.

To define a `std::string` base type for xsd:QName, we use a typedef:

    typedef std::string xsd__QName;

The `xsd__QName` string content is normalized, just as with the `_QName`
normalization.

To serialize strings that contain literal XML content to be reproduced in the
XML value space, use the built-in `_XML` string type, which is a regular C
string type (`char*`) that maps to plain XML CDATA.

To define a `std::string` base type for literal XML content, use a typedef:

    typedef std::string XML;

Strings can hold any of the values of the XSD built-in primitive types.  We can
use a string typedef to declare the use of the string type as a XSD built-in
type:

    typedef std::string xsd__token;

We MUST ensure that the string values we populate in this type conform to the
XML standard, which in case of xsd:token is: the lexical and value spaces of
xsd:token are the sets of all strings after whitespace replacement of any
occurrence of `#x9`, `#xA` , and `#xD` by `#x20` and collapsing.

To copy `char*` or `wchar_t*` strings with a context that manages the allocated
memory, use functions

    char *soap_strdup(struct soap*, const char*)
    wchar_t *soap_wstrdup(struct soap*, const wchar_t*)

To convert a wide string to a UTF-8 encoded string, use function

    const char* SOAP_FMAC2 soap_wchar2s(struct soap*, const wchar_t *s)

The function allocates and returns a string, with its memory being managed by
the context.

To convert a UTF-8 encoded string to a wide string, use function

    int soap_s2wchar(struct soap*, const char *from, wchar_t **to, long minlen, long maxlen)

where `to` is set to point to an allocated `wchar_t*` string.  Pass `-1` for
`minlen` and `maxlen` to ignore length constraints on the target string.  The
function returns `SOAP_OK` or an error when the length constraints are not met.


Date and time types                                                    {#toxsd7}
-------------------

The C/C++ `time_t` type is mapped to the built-in xsd:dateTime XSD type that
represents a date and time within a time zone (typically UTC).

The XML value space contains ISO 8601 Gregorian time instances of the form
`[-]CCYY-MM-DDThh:mm:ss.sss[Z|(+|-)hh:mm]`, where `Z` is the UTC time zone or a
time zone offset `(+|-)hh:mm]` from UTC is used.

A `time_t` value is considered and represented in UTC by the serializer.

Because the `time_t` value range is restricted to dates after 01/01/1970, care
must be taken to ensure the range of xsd:dateTime values in XML exchanges do
not exceed the `time_t` range.

This restriction does not hold for `struct tm` (`<time.h>`), which we can use
to store and exchange a date and time in UTC without date range restrictions.
The serializer uses the `struct tm` members directly for the XML value space of
xsd:dateTime:

    struct tm
    {
      int    tm_sec;    // seconds (0 - 60)
      int    tm_min;    // minutes (0 - 59)
      int    tm_hour;   // hours (0 - 23)
      int    tm_mday;   // day of month (1 - 31)
      int    tm_mon;    // month of year (0 - 11)
      int    tm_year;   // year - 1900
      int    tm_wday;   // day of week (Sunday = 0) (NOT USED)
      int    tm_yday;   // day of year (0 - 365) (NOT USED)
      int    tm_isdst;  // is summer time in effect?
      char*  tm_zone;   // abbreviation of timezone (NOT USED)
    };

You will lose the day of the week information.  It is always Sunday
(`tm_wday=0`) and the day of the year is not set either.  The time zone is UTC.

This `struct tm` type is mapped to the built-in xsd:dateTime XSD type and
serialized with the custom serializer `custom/struct_tm.h` that declares a
`xsd__dateTime` type:

    #import "custom/struct_tm.h"  // import typedef struct tm xsd__dateTime;
    ... use xsd__dateTime ...

Compile and link your code with `custom/struct_tm.c`.

The `struct timeval` (`<sys/time.h>`) type is mapped to the built-in
xsd:dateTime XSD type and serialized with the custom serializer
`custom/struct_timeval.h` that declares a `xsd__dateTime` type:

    #import "custom/struct_timeval.h"  // import typedef struct timeval xsd__dateTime;
    ... use xsd__dateTime ...

Compile and link your code with `custom/struct_timeval.c`.

Note that the same value range restrictions apply to `struct timeval` as they
apply to `time_t`.  The added benefit of `struct timeval` is the addition of
a microsecond-precise clock:

    struct timeval
    {
      time_t       tv_sec;   // seconds since Jan. 1, 1970
      suseconds_t  tv_usec;  // and microseconds
    };

A C++11 `std::chrono::system_clock::time_point` type is mapped to the built-in
xsd:dateTime XSD type and serialized with the custom serializer
`custom/chrono_time_point.h` that declares a `xsd__dateTime` type:

    #import "custom/chrono_time_point.h"  // import typedef std::chrono::system_clock::time_point xsd__dateTime;
    ... use xsd__dateTime ...

Compile and link your code with `custom/chrono_time_point.cpp`.

The `struct tm` type is mapped to the built-in xsd:date XSD type and serialized
with the custom serializer `custom/struct_tm_date.h` that declares a
`xsd__date` type:

    #import "custom/struct_tm_date.h"  // import typedef struct tm xsd__date;
    ... use xsd__date ...

Compile and link your code with `custom/struct_tm_date.c`.

The XML value space of xsd:date are Gregorian calendar dates of the form
`[-]CCYY-MM-DD[Z|(+|-)hh:mm]` with a time zone.

The serializer ignores the time part and the deserializer only populates the
date part of the struct, setting the time to 00:00:00.  There is no unreasonable
limit on the date range because the year field is stored as an integer (`int`).

An `unsigned long long` (`ULONG64` or `uint64_t`) type that contains a 24 hour
time in microseconds UTC is mapped to the built-in xsd:time XSD type and
serialized with the custom serializer `custom/long_time.h` that declares a
`xsd__time` type:

    #import "custom/long_time.h"  // import typedef unsigned long long xsd__time;
    ... use xsd__time ...

Compile and link your code with `custom/long_time.c`.

This type represents 00:00:00.000000 to 23:59:59.999999, from `0` to an upper
bound of `86399999999`.  A microsecond resolution means that a 1 second
increment requires an increment of 1000000 in the integer value.

The XML value space of xsd:time are points in time recurring each day of the
form `hh:mm:ss.sss[Z|(+|-)hh:mm]`, where `Z` is the UTC time zone or a time
zone offset from UTC is used.  The `xsd__time` value is always considered and
represented in UTC by the serializer.

To convert date and/or time values to a string, we use the auto-generated
function for type `T`:

    const char *soap_T2s(struct soap*, T val)

For date and time types `T`, the string returned is stored in an internal
buffer, so you MUST copy it to keep it from being overwritten.  For example,
use `soap_strdup(struct soap*, const char*)`.

To convert a string to a date/time value, we use the auto-generated function

    int soap_s2T(struct soap*, const char *str, T *val)
  
where `T` is for example `dateTime` (for `time_t`), `xsd__dateTime` (for
`struct tm`, `struct timeval`, or `std::chrono::system_clock::time_point`).
The function `soap_s2T` returns `SOAP_OK` on success or an error when the value
is not a date/time.


Time duration types                                                    {#toxsd8}
-------------------

The XML value space of xsd:duration are values of the form `PnYnMnDTnHnMnS`
where the capital letters are delimiters.  Delimiters may be omitted when the
corresponding member is not used.

A `long long` (`LONG64` or `int64_t`) type that contains a duration (time
lapse) in milliseconds is mapped to the built-in xsd:duration XSD type and
serialized with the custom serializer `custom/duration.h` that declares a
`xsd__duration` type:

    #import "custom/duration.h"  // import typedef long long xsd__duration;
    ... use xsd__duration ...

Compile and link your code with `custom/duration.c`.

The duration type `xsd__duration` can represent 106,751,991,167 days forward
and backward with millisecond precision.

Durations that exceed a month are always output in days, rather than months to
avoid days-per-month conversion inacurracies.

Durations that are received in years and months instead of total number of days
from a reference point are not well defined, since there is no accepted
reference time point (it may or may not be the current time).  The decoder
simple assumes that there are 30 days per month.  For example, conversion of
"P4M" gives 120 days.  Therefore, the durations "P4M" and "P120D" are assumed
to be identical, which is not necessarily true depending on the reference point
in time.

Rescaling of the duration value by may be needed when adding the duration value
to a `time_t` value, because `time_t` may or may not have a seconds resolution,
depending on the platform and possible changes to `time_t`.

Rescaling is done automatically when you add a C++11 `std::chrono::nanoseconds`
value to a `std::chrono::system_clock::time_point` value.  To use
`std::chrono::nanoseconds` as xsd:duration:

    #import "custom/chrono_duration.h"  // import typedef std::chrono::duration xsd__duration;
    ... use xsd__duration ...

Compile and link your code with `custom/chrono_duration.cpp`.

This type can represent 384,307,168 days (2^63 nanoseconds) forwards and
backwards in time in increments of 1 ns (1/1000000000 second).

The same observations with respect to receiving durations in years and months
apply to this serializer's decoder.

To convert duration values to a string, we use the auto-generated function

    const char *soap_xsd__duration2s(struct soap*, xsd__duration val)

The string returned is stored in an internal buffer, so you MUST copy it to
keep it from being overwritten,  Use `soap_strdup(struct soap*, const char*)`
for example,

To convert a string to a duration value, we use the auto-generated function

    int soap_s2xsd__dateTime(struct soap*, const char *str, xsd__dateTime *val)

The function returns `SOAP_OK` on success or an error when the value is not a
duration.


Classes and structs                                                    {#toxsd9}
-------------------

Classes and structs are mapped to XSD complexTypes.  The XML value space
consists of XML elements with attributes and subelements, possibly constrained
by validation rules that enforce element and attribute occurrence contraints,
numerical value range constraints, and string length and pattern constraints.

Classes that are declared with the gSOAP tools are limited to single
inheritence only.  Structs cannot be inherited.

The class and struct name is bound to an XML namespace by means of the prefix
naming convention or by using [Colon notation](#toxsd1):

    //gsoap ns schema namespace: urn:types
    class ns__record
    {
     public:
      std::string  name;
      uint64_t     SSN;
      ns__record  *spouse;
      ns__record();
      ~ns__record();
     protected:
      struct soap  *soap;
    };

In the example above, we also added a context pointer to the `struct soap` that
manages this instance.  It is set when the instance is created in the engine's
context, for example when deserialized and populated by the engine.

The class maps to a complexType in the soapcpp2-generated schema:

    <complexType name="record">
      <sequence>
        <element name="name" type="xsd:string" minOccurs="1" maxOccurs="1"/>
        <element name="SSN" type="xsd:unsignedLong" minOccurs="1" maxOccurs="1"/>
        <element name="spouse" type="ns:record" minOccurs="0" maxOccurs="1" nillable="true"/>
      </sequence>
    </complexType>

### Serializable versus transient types and members                  {#toxsd9-1}

Public data members of a class or struct are serialized.  Private and protected
members are transient and not serializable.

Also `const` and `static` members are not serializable, with the exception of
`const char*` and `const wchar_t*`.

Types and specific class/struct members can be made transient by using the
`extern` qualifier:

    extern class std::ostream;  // declare 'std::ostream' transient
    class ns__record
    {
     public:
      extern int       num;         // not serialized
      std::ostream     out;         // not serialized
      static const int MAX = 1024;  // not serialized
    };

By declaring `std::ostream` transient we can use this type where we need it and
without soapcpp2 complaining that this class is not defined.

### Volatile classes and structs                                     {#toxsd9-2}

Classes and structs can be declared `volatile` with the gSOAP tools.  This means
that they are already declared elsewhere in our project's source code.  We do
not want soapcpp2 to generate a second definition for these types.

For example, `struct tm` is declared in `<time.h>`.  We want it serializable and
serialize only a selection of its data members:

    volatile struct tm
    {
      int    tm_sec;    // seconds (0 - 60)
      int    tm_min;    // minutes (0 - 59)
      int    tm_hour;   // hours (0 - 23)
      int    tm_mday;   // day of month (1 - 31)
      int    tm_mon;    // month of year (0 - 11)
      int    tm_year;   // year - 1900
    };

We can declare classes and structs `volatile` for any such types we want to
serialize by only providing the public data members we want to serialize.

Colon notation comes in handy to bind an existing class or struct to a schema.
For example, we can change the `tm` name as follows without affecting the code
that uses `struct tm` generated by soapcpp2:

    volatile struct ns:tm { ... }

This struct maps to a complexType in the soapcpp2-generated schema:

    <complexType name="tm">
      <sequence>
        <element name="tm-sec" type="xsd:int" minOccurs="1" maxOccurs="1"/>
        <element name="tm-min" type="xsd:int" minOccurs="1" maxOccurs="1"/>
        <element name="tm-hour" type="xsd:int" minOccurs="1" maxOccurs="1"/>
        <element name="tm-mday" type="xsd:int" minOccurs="1" maxOccurs="1"/>
        <element name="tm-mon" type="xsd:int" minOccurs="1" maxOccurs="1"/>
        <element name="tm-year" type="xsd:int" minOccurs="1" maxOccurs="1"/>
      </sequence>
    </complexType>

### Mutable classes and structs                                      {#toxsd9-3}

Classes and structs can be declared `mutable` with the gSOAP tools.  This means
that their definition can be spread out over the source code.  This promotes the
concept of a class or struct as a *row of named values*, also known as a *named
tuple*, that can be extended at compile time in our source code with additional
members.  Because these types differ from the traditional object-oriented
principles and design concepts of classes and objects, constructors and
destructors cannot be defined (also because we cannot guarantee merging these
into one such that all members will be initialized).  A default constructor,
copy constructor, assignment operation, and destructor will be assigned
automatically by soapcpp2.

    mutable struct ns__tuple
    {
      @std::string  id;
    };

    mutable struct ns__tuple
    {
      std::string  name;
      std::string  value;
    };

The members are collected into one definition generated by soapcpp2.  Members
may be repeated from one definition to another, but only if their associated
types are identical.  So, for example, a third extension with a `value` member
with a different type fails:

    mutable struct ns__tuple
    {
      float        value;  // BAD: value is already declared std::string
    };

The `mutable` concept has proven to be very useful when declaring and
collecting SOAP Headers for multiple services, which are collected into one
`struct SOAP_ENV__Header` by the soapcpp2 tool.

### Default member values in C and C++                               {#toxsd9-4}

Class and struct data members in C and C++ may be declared with an optional
default initialization value that is provided "inline" with the declaration of
the member:

    class ns__record
    {
     public:
      std::string name = "Joe";

These initializations are made by the default constructor that is added by
soapcpp2 to each class and struct.  A constructor is only added when a default
constructor is not already defined with the class declaration.  You can
explicitly (re)initialize an object with the auto-generated
`soap_default(struct soap*)` method of a class and the auto-generated
`soap_default_T(struct soap*, T*)` function for a struct `T` in C and C++.

Initializations can only be provided for members that have primitive types
(`bool`, `enum`, `time_t`, numeric and string types).

### Attribute members                                                {#toxsd9-5}

Class and struct data members can be declared as XML attributes by annotating
their type with a `@` with the declaration of the member:

    class ns__record
    {
     public:
      @std::string name;
      @uint64_t    SSN;
      ns__record  *spouse;
    };

This class maps to a complexType in the soapcpp2-generated schema:

    <complexType name="record">
      <sequence>
        <element name="spouse" type="ns:record" minOccurs="0" maxOccurs="1" nillable="true"/>
      </sequence>
      <attribute name="name" type="xsd:string" use="required"/>
      <attribute name="SSN" type="xsd:unsignedLong" use="required"/>
    </complexType>

An example XML instance of `ns__record` is:

    <ns:record xmlns:ns="urn:types" name="Joe" SSN="1234567890">
      <spouse>
        <name>Jane</name>
        <SSN>1987654320</SSN>
      </spouse>
    </ns:record>

Attribute data members are restricted to primitive types (`bool`, `enum`,
`time_t`, numeric and string types), `xsd__hexBinary`, `xsd__base64Binary`, and
custom serializers, such as `xsd__dateTime`.  Custom serializers for types that
may be used as attributes MUST define `soap_s2T` and `soap_T2s` functions that
convert values of type `T` to strings and back.

Attribute data members can be pointers and smart pointers to these types, which
permits attributes to be optional.

### (Smart) pointer members and their occurrence constraints         {#toxsd9-6}

A public pointer-typed data member is serialized by following its (smart)
pointer(s) to the value pointed to.

Pointers that are NULL and smart pointers that are empty are serialized to
produce omitted element and attribute values, unless an element is required
and is nillable.

To control the occurrence requirements of pointer-based data members,
occurrence constraints are associated with data members in the form of a range
`minOccurs : maxOccurs`.  For non-repeatable (meaning, not a container or array)
data members, there are only three reasonable occurrence constraints:

- `0:0` means that this element or attribute is prohibited.
- `0:1` means that this element or attribute is optional.
- `1:1` means that this element or attribute is required.

Pointer-based data members have a default `0:1` occurrence constraint, making
them optional, and their XSD schema local element/attribute definition is
marked as nillable.  Non-pointer data members have a default `1:1` occurence
constraint, making them required.

A pointer data member that is explicitly marked as required with `1:1` will be
serialized as an element with an xsi:nil attribute, thus effectively revealing
the NULL property of its value.

A non-pointer data member that is explicitly marked as optional with `0:1` will
be set to its default value when no XML value is presented to the deserializer.
A default value can be assigned to data members that have primitive types.

Consider for example:

    class ns__record
    {
     public:
      std::shared_ptr<std::string>  name;              // optional (0:1)
      uint64_t                      SSN    0:1 = 999;  // forced this to be optional with default 999
      ns__record                   *spouse 1:1;        // forced this to be required (only married people)
    };

This class maps to a complexType in the soapcpp2-generated schema:

    <complexType name="record">
      <sequence>
        <element name="name" type="xsd:string" minOccurs="0" maxOccurs="1" nillable="true"/>
        <element name="SSN" type="xsd:unsignedLong" minOccurs="0" maxOccurs="1" default="999"/>
        <element name="spouse" type="ns:record" minOccurs="1" maxOccurs="1" nillable="true"/>
      </sequence>
    </complexType>

An example XML instance of `ns__record` with its `name` string value set to
`Joe`, `SSN` set to its default, and `spouse` set to NULL:

    <ns:record xmlns:ns="urn:types">
      <name>Joe</name>
      <SSN>999</SSN>
      <spouse xsi:nil="true"/>
    </ns:record>

@note In general, a smart pointer is simply declared as a `volatile` template
in a gSOAP header file for soapcpp2:

    volatile template <class T> class NAMESPACE::shared_ptr;

@note The soapcpp2 tool generates code that uses `NAMESPACE::shared_ptr` and
`NAMESPACE::make_shared` to create shared pointers to objects, where
`NAMESPACE` is any valid C++ namespace such as `std` and `boost` if you have
Boost installed.

### Container members and their occurrence constraints               {#toxsd9-7}

Class and struct data member types that are containers `std::deque`,
`std::list`, `std::vector` and `std::set` are serialized as a collections
of values.

You can use `std::deque`, `std::list`, `std::vector`, and `std::set` containers by importing:

    #import "import/stl.h"        // import all containers
    #import "import/stldeque.h"   // import deque
    #import "import/stllist.h"    // import list
    #import "import/stlvector.h"  // import vector
    #import "import/stlset.h"     // import set

For example, to use a vector data mamber to store names in a record:

    #import "import/stlvector.h"
    class ns__record
    {
     public:
      std::vector<std::string>  names;
      uint64_t                  SSN;
    };

To limit the number of names in the vector within reasonable bounds, occurrence
constraints are associated with the container.  Occurrence constraints are of
the form `minOccurs : maxOccurs`:

    #import "import/stlvector.h"
    class ns__record
    {
     public:
      std::vector<std::string>  names 1:10;
      uint64_t                  SSN;
    };

This class maps to a complexType in the soapcpp2-generated schema:

    <complexType name="record">
      <sequence>
        <element name="name" type="xsd:string" minOccurs="1" maxOccurs="10"/>
        <element name="SSN" type="xsd:unsignedLong" minOccurs="1" maxOccurs="1""/>
      </sequence>
    </complexType>

@note In general, a container is simply declared as a template in a gSOAP
header file for soapcpp2.  All class templates are considered containers
(except when declared `volatile`, see smart pointers).  For example,
`std::vector` is declared in `gsoap/import/stlvector.h` as:

    template <class T> class std::vector;

@note You can define and use your own containers.  The soapcpp2 tool generates
code that uses the following members of the `template <typename T> class C`
container:

        void              C::clear()
        C::iterator       C::begin()
        C::const_iterator C::begin() const
        C::iterator       C::end()
        C::const_iterator C::end() const
        size_t            C::size() const
        C::iterator       C::insert(C::iterator pos, const T& val)

@note For more details see the example `simple_vector` container with
documentation in the package under `gsoap/samples/template`.

Because C does not support a container template library, we can use a
dynamically-sized array of values.  This array is declared as a size-pointer
member pair:

    struct ns__record
    {
      $int      sizeofnames;  // array size
      char*    *names;        // array of char* names
      uint64_t  SSN;
    };

where the marker `$` with `int` denotes a special type that is used to store
the array size and to indicate that this is a size-pointer member pair that
declares a dynamically-sized array.

This class maps to a complexType in the soapcpp2-generated schema:

    <complexType name="record">
      <sequence>
        <element name="name" type="xsd:string" minOccurs="0" maxOccurs="unbounded" nillable="true"/>
        <element name="SSN" type="xsd:unsignedLong" minOccurs="1" maxOccurs="1""/>
      </sequence>
    </complexType>

To limit the number of names in the array within reasonable bounds, occurrence
constraints are associated with the array size member.  Occurrence constraints
are of the form `minOccurs : maxOccurs`:

    struct ns__record
    {
      $int      sizeofnames 1:10;  // array size 1..10
      char*    *names;             // array of one to ten char* names
      uint64_t  SSN;
    };

This class maps to a complexType in the soapcpp2-generated schema:

    <complexType name="record">
      <sequence>
        <element name="name" type="xsd:string" minOccurs="1" maxOccurs="10" nillable="true"/>
        <element name="SSN" type="xsd:unsignedLong" minOccurs="1" maxOccurs="1""/>
      </sequence>
    </complexType>

### Union members                                                    {#toxsd9-8}

A union member in a class or in a struct cannot be serialized unless a
discriminating *variant selector* member is provided that tells the serializer
which union field to serialize.  This effectively creates a *tagged union*.

The variant selector is associated with the union as a selector-union member
pair, where the variant selector is a special `$int` member:

    class ns__record
    {
     public:
      $int  xORnORs;  // variant selector
      union choice
      {
        float x;
        int   n;
        char *s;
      } u;
      std::string name;
    };

The variant selector values are auto-generated based on the union name `choice`
and the names of its members `x`, `n`, and `s`:

- `xORnORs = SOAP_UNION_choice_x` when `u.x` is valid.
- `xORnORs = SOAP_UNION_choice_n` when `u.n` is valid.
- `xORnORs = SOAP_UNION_choice_s` when `u.s` is valid.
- `xORnORs = 0` when none are valid (should only be used with great care,
  because XML content validation may fail when content is required but absent).

This class maps to a complexType with a sequence and choice in the
soapcpp2-generated schema:

    <complexType name="record">
      <sequence>
        <choice>
          <element name="x" type="xsd:float" minOccurs="1" maxOccurs="1"/>
          <element name="n" type="xsd:int" minOccurs="1" maxOccurs="1"/>
          <element name="s" type="xsd:string" minOccurs="0" maxOccurs="1" nillable="true"/>
        </choice>
        <element name="names" type="xsd:string" minOccurs="1" maxOccurs="10" nillable="true"/>
      </sequence>
    </complexType>

### Adding get and set methods                                       {#toxsd9-9}

A public `get` method may be added to a class or struct, which will be
triggered by the deserializer.  This method will be invoked right after the
instance is populated by the deserializer.  The `get` method can be used to
update or verify deserialized content.  It should return `SOAP_OK` or set
`soap::error` to a nonzero error code and return it.

A public `set` method may be added to a class or struct, which will be
triggered by the serializer.  The method will be invoked just before the
instance is serialized.  Likewise, the `set` method should return `SOAP_OK` or
set set `soap::error` to a nonzero error code and return it.

For example, adding a `set` and `get` method to a class declaration:

    class ns__record
    {
     public:
      int set(struct soap*);  // triggered before serialization
      int get(struct soap*);  // triggered after deserialization

To add these and othe rmethods to classes and structs with wsdl2h and
`typemap.dat`, please see [Class/struct member additions](#typemap3).

### Defining document root elements                                 {#toxsd9-10}

To define and reference XML document root elements we use type names that start
with an underscore:

    class _ns__record

Alternatively, we can use a typedef to define a document root element with a
given type:

    typedef ns__record _ns__record;

This typedef maps to a global root element that is added to the
soapcpp2-generated schema:

    <element name="record" type="ns:record"/>

An example XML instance of `_ns__record` is:

    <ns:record xmlns:ns="urn:types">
      <name>Joe</name>
      <SSN>1234567890</SSN>
      <spouse>
        <name>Jane</name>
        <SSN>1987654320</SSN>
      </spouse>
    </ns:record>

Global-level element/attribute definitions are also referenced and/or added to
the generated schema when serializable data members reference these by their
qualified name:

    typedef std::string _ns__name 1 : 100;
    class _ns__record
    {
     public:
      @_QName      xsi__type;  // built-in XSD attribute xsi:type
      _ns__name    ns__name;   // ref to global ns:name element
      uint64_t     SSN;
      _ns__record *spouse;
    };

These types map to the following comonents in the soapcpp2-generated schema:

    <simpleType name="name">
      <restriction base="xsd:string">
        <minLength value="1"/>
        <maxLength value="100"/>
      </restriction>
    </simpleType>
    <element name="name" type="ns:name"/>
    <complexType name="record">
      <sequence>
        <element ref="ns:name" minOccurs="1" maxOccurs="1"/>
        <element name="SSN" type="xsd:unsignedLong" minOccurs="1" maxOccurs="1"/>
        <element name="spouse" type="ns:record" minOccurs="0" maxOccurs="1" nillable="true"/>
      </sequence>
      <attribute ref="xsi:type" use="optional"/>
    </complexType>
    <element name="record" type="ns:record"/>

@warning Use only use qualified member names when their types match the root
element types that they refer to.  For example:

    class _ns__record
    {
     public:
      int  ns__name;  // BAD: element ns:name is NOT of an int type

@warning Therefore, we recommend to avoid qualified member names and only use
them when referring to standard XSD elements and attributes, such as
`xsi__type`, and `xsd__lang`.  The soapcpp2 tool does not prevent abuse of this
mechanism.

### Operations on classes and structs                               {#toxsd9-11}

The following functions/macros are generated by soapcpp2 for each type `T`,
which should make it easier to send, receive, and copy XML data in C and in
C++:

- `int soap_write_T(struct soap*, T*)` writes an instance of `T` to a FILE (via
   `FILE *soap::sendfd)`) or to a stream (via `std::ostream *soap::os`).
   Returns `SOAP_OK` on success or an error code, also stored in `soap->error`.

- `int soap_read_T(struct soap*, T*)` reads an instance of `T` from a FILE (via
   `FILE *soap::recvfd)`) or from a stream (via `std::istream *soap::is`).
   Returns `SOAP_OK` on success or an error code, also stored in `soap->error`.

- `void soap_default_T(struct soap*, T*)` sets an instance `T` to its default
  value, resetting members of a struct to their initial values (for classes we
  use method `T::soap_default`, see below).

- `T * soap_dup_T(struct soap*, T *dst, const T *src)` (soapcpp2 option `-Ec`)
  deep copy `src` into `dst`, replicating all deep cycles and shared pointers
  when a managing soap context is provided as argument.  When `dst` is NULL,
  allocates space for `dst`.  Deep copy is a tree when argument is NULL, but the
  presence of deep cycles will lead to non-termination.  Use flag
  `SOAP_XML_TREE` with managing context to copy into a tree without cycles and
  pointers to shared objects.  Returns `dst` (or allocated space when `dst` is
  NULL).

- `void soap_del_T(const T*)` (soapcpp2 option `-Ed`) deletes all
  heap-allocated members of this object by deep deletion ONLY IF this object
  and all of its (deep) members are not managed by a soap context AND the deep
  structure is a tree (no cycles and co-referenced objects by way of multiple
  (non-smart) pointers pointing to the same data).  Can be safely used after
  `soap_dup(NULL)` to delete the deep copy.  Does not delete the object itself.

When in C++ mode, soapcpp2 tool adds several methods to classes and structs, in
addition to adding a default constructor and destructor (when these were not
explicitly declared).

The public methods added to a class/struct `T`:

- `virtual int T::soap_type(void)` returns a unique type ID (`SOAP_TYPE_T`).
  This numeric ID can be used to distinguish base from derived instances.

- `virtual void T::soap_default(struct soap*)` sets all data members to
  default values.

- `virtual void T::soap_serialize(struct soap*) const` serializes object to
  prepare for SOAP 1.1/1.2 encoded output (or with `SOAP_XML_GRAPH`) by
  analyzing its (cyclic) structures.

- `virtual int T::soap_put(struct soap*, const char *tag, const char *type) const`
  emits object in XML, compliant with SOAP 1.1 encoding style, return error
  code or `SOAP_OK`.  Requires `soap_begin_send(soap)` and
  `soap_end_send(soap)`.

- `virtual int T::soap_out(struct soap*, const char *tag, int id, const char *type) const`
  emits object in XML, with tag and optional id attribute and xsi:type, return
  error code or `SOAP_OK`.  Requires `soap_begin_send(soap)` and
  `soap_end_send(soap)`.

- `virtual void * T::soap_get(struct soap*, const char *tag, const char *type)`
  Get object from XML, compliant with SOAP 1.1 encoding style, return pointer
  to object or NULL on error.  Requires `soap_begin_recv(soap)` and
  `soap_end_recv(soap)`.

- `virtual void *soap_in(struct soap*, const char *tag, const char *type)`
  Get object from XML, with matching tag and type (NULL matches any tag and
  type), return pointer to object or NULL on error.  Requires
  `soap_begin_recv(soap)` and `soap_end_recv(soap)`

- `virtual T * T::soap_alloc(void) const` returns a new object of type `T`,
  default initialized and not managed by a soap context.

- `virtual T * T::soap_dup(struct soap*) const` (soapcpp2 option `-Ec`) returns
  a duplicate of this object by deep copying, replicating all deep cycles and
  shared pointers when a managing soap context is provided as argument.  Deep
  copy is a tree when argument is NULL, but the presence of deep cycles will
  lead to non-termination.  Use flag `SOAP_XML_TREE` with the managing context
  to copy into a tree without cycles and pointers to shared objects.

- `virtual void T::soap_del() const` (soapcpp2 option `-Ed`) deletes all
  heap-allocated members of this object by deep deletion ONLY IF this object
  and all of its (deep) members are not managed by a soap context AND the deep
  structure is a tree (no cycles and co-referenced objects by way of multiple
  (non-smart) pointers pointing to the same data).  Can be safely used after
  `soap_dup(NULL)` to delete the deep copy.  Does not delete the object itself.


Special classes and structs                                           {#toxsd10}
---------------------------

### SOAP encoded arrays                                             {#toxsd10-1}

A class or struct with the following layout is a one-dimensional SOAP encoded
Array type:

    class ArrayOfT
    {
     public:
      T   *__ptr;   // array pointer
      int  __size;  // array size
    };

where `T` is the array element type.  A multidimensional SOAP Array is:

    class ArrayOfT
    {
     public:
      T   *__ptr;      // array pointer
      int  __size[N];  // array size of each dimension
    };

where `N` is the constant number of dimensions.  The pointer points to an array
of `__size[0]*__size[1]* ... * __size[N-1]` elements.

This maps to a complexType restriction of SOAP-ENC:Array in the
soapcpp2-generated schema:

    <complexType name="ArrayOfT">
      <complexContent>
        <restriction base="SOAP-ENC:Array">
          <sequence>
            <element name="item" type="T" minOccurs="0" maxOccurs="unbounded" nillable="true"/>
          </sequence>
          <attribute ref="SOAP-ENC:arrayType" WSDL:arrayType="ArrayOfT[]"/>
        </restriction>
      </complexContent>
    </complexType>

The name of the class can be arbitrary.  We often use `ArrayOfT` without a
prefix to distinguish arrays from other classes and structs.

With SOAP 1.1 encoding, an optional offset member can be added that controls
the start of the index range for each dimension:

    class ArrayOfT
    {
     public:
      T   *__ptr;        // array pointer
      int  __size[N];    // array size of each dimension
      int  __offset[N];  // array offsets to start each dimension
    };

For example, we can define a matrix of floats as follows:

    class Matrix
    {
     public:
      double *__ptr;
      int     __size[2];
    };

The following code populates the matrix and serializes it in XML:

    soap *soap = soap_new1(SOAP_XML_INDENT);
    Matrix A;
    double a[6] = { 1, 2, 3, 4, 5, 6 };
    A.__ptr = a;
    A.__size[0] = 2;
    A.__size[1] = 3;
    soap_write_Matrix(soap, &A);

Matrix A is serialized as an array with 2x3 values:

    <SOAP-ENC:Array SOAP-ENC:arrayType="xsd:double[2,3]" ...>
      <item>1</item>
      <item>2</item>
      <item>3</item>
      <item>4</item>
      <item>5</item>
      <item>6</item>
    </SOAP-ENC:Array>

### XSD hexBinary and base64Binary types                            {#toxsd10-2}

A special case of a one-dimensional array is used to define xsd:hexBinary and
xsd:base64Binary types when the pointer type is `unsigned char`:

    class xsd__hexBinary
    {
     public:
      unsigned char *__ptr;   // points to raw binary data
      int            __size;  // size of data
    };

and

    class xsd__base64Binary
    {
     public:
      unsigned char *__ptr;   // points to raw binary data
      int            __size;  // size of data
    };

### MIME/MTOM attachment binary types                               {#toxsd10-3}

A class or struct with a binary content layout can be extended to support
MIME/MTOM (and older DIME) attachments, such as in xop:Include elements:

    //gsoap xop schema import: http://www.w3.org/2004/08/xop/include
    class _xop__Include
    {
     public:
      unsigned char *__ptr;   // points to raw binary data
      int            __size;  // size of data
      char          *id;      // NULL to generate an id, or set to a unique UUID
      char          *type;    // MIME type of the data
      char          *options; // optional description of MIME attachment
    };

Attachments are beyond the scope of this document and we refer to the gSOAP
user guide for more details.

### Wrapper class/struct for simpleContent                          {#toxsd10-4}

A class or struct with the following layout is a complexType that wraps
simpleContent:

    class ns__simple
    {
     public:
      T   __item;
    };

The type `T` is a primitive type (`bool`, `enum`, `time_t`, numeric and string
types), `xsd__hexBinary`, `xsd__base64Binary`, and custom serializers, such as
`xsd__dateTime`.

This maps to a complexType with simpleContent in the soapcpp2-generated schema:

    <complexType name="simple">
      <simpleContent>
        <extension base="T"/>
      </simpleContent>
    </complexType> 

A wrapper class/struct may include any number of attributes declared with `@`.


Serialization rules                                                     {#rules}
===================

A presentation on XML data bindings is not complete without discussing the
serialization rules that put your data in XML on the wire.

There are several options to choose from to serialize data in XML.  The choice
depends on the use of the SOAP protocol or if SOAP is not required.  The wsdl2h
tool automates this for you by taking the WSDL transport bindings into account
when generating the service functions in C and C++ that use SOAP or REST.

The gSOAP tools are not limited to SOAP.  The tools implement generic XML data
bindings for SOAP, REST, and other uses of XML.  So you can read and write XML
using the serializing [Operations on classes and structs](#toxsd9-11).

The following sections briefly explain the serialization rules with respect to
the SOAP protocol for XML Web services.  A basic understanding of the SOAP
protocol is useful when developing client and server applications that must
interoperate with other SOAP applications.

SOAP/REST Web service client and service operations are represented as
functions in our gSOAP header file for soapcpp2.  The soapcpp2 tool will
translate these function to client-side service invocation calls and
server-side service operation dispatchers.

A discussion of SOAP clients and servers is beyond the scope of this document.
However, the SOAP options discussed here also apply to SOAP client and server
development.


SOAP document versus rpc style                                        {#doc-rpc}
------------------------------

The `wsdl:binding/soap:binding/@style` attribute in the wsdl:binding section of
a WSDL is either "document" or "rpc".  The "rpc" style refers to SOAP RPC
(Remote Procedure Call), which is more restrictive than the "document" style by
requiring one XML element in the SOAP Body to act as the procedure name with
XML subelements as its parameters.

For example, the following directives in the gSOAP header file for soapcpp2
declare that `DBupdate` is a SOAP RPC encoding service method:

    //gsoap ns service namespace:       urn:DB
    //gsoap ns service method-protocol: DBupdate SOAP
    //gsoap ns service method-style:    DBupdate rpc
    int ns__DBupdate(...);

The XML payload has a SOAP envelope, optional SOAP header, and a SOAP body with
one element representing the operation with the parameters as subelements:

    <SOAP-ENV:Envelope
      xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
      xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:xsd="http://www.w3.org/2001/XMLSchema"
      xmlsn:ns="urn:DB">
      <SOAP-ENV:Body>
        <ns:DBupdate>
          ...
        </ns:DBupdate>
      </SOAP-ENV:Body>
    </SOAP-ENV:Envelope>

The "document" style puts no restrictions on the SOAP Body content.  However, we
recommend that the first element's tag name in the SOAP Body should be unique
to each type of operation, so that the receiver can dispatch the operation
based on this element's tag name.  Alternatively, the HTTP URL path can be used
to specify the operation, or the HTTP action header can be used to dispatch
operations automatically on the server side (soapcpp2 options -a and -A).


SOAP literal versus encoding                                          {#lit-enc}
----------------------------

The `wsdl:operation/soap:body/@use` attribute in the wsdl:binding section of a
WSDL is either "literal" or "encoded".  The "encoded" use refers to the SOAP
encoding rules that support id-ref multi-referenced elements to serialize
data as graphs.

SOAP encoding is very useful if the data internally forms a graph (including
cycles) and we want the graph to be serialized in XML in a format that ensures
that its structure is preserved.  In that case, SOAP 1.2 encoding is the best
option.

SOAP encoding also adds encoding rules for [SOAP arrays](toxsd10) to serialize
multi-dimensional arrays.  The use of XML attributes to exchange XML data in
SOAP encoding is not permitted.  The only attributes permitted are the standard
XSD attributes, SOAP encoding attributes (such as for arrays), and id-ref.

For example, the following directives in the gSOAP header file for soapcpp2
declare that `DBupdate` is a SOAP RPC encoding service method:

    //gsoap ns service namespace:       urn:DB
    //gsoap ns service method-protocol: DBupdate SOAP
    //gsoap ns service method-style:    DBupdate rpc
    //gsoap ns service method-encoding: DBupdate encoded
    int ns__DBupdate(...);

The XML payload has a SOAP envelope, optional SOAP header, and a SOAP body with
an encodingStyle attribute for SOAP 1.1 encoding and an element representing the
operation with parameters that are SOAP 1.1 encoded:

    <SOAP-ENV:Envelope
      xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
      xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:xsd="http://www.w3.org/2001/XMLSchema"
      xmlsn:ns="urn:DB">
      <SOAP-ENV:Body SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
        <ns:DBupdate>
          <records SOAP-ENC:arrayType="ns:record[3]">
            <item>
              <name href="#_1"/>
              <SSN>1234567890</SSN>
            </item>
            <item>
              <name>Jane</name>
              <SSN>1987654320</SSN>
            </item>
            <item>
              <name href="#_1"/>
              <SSN>2345678901</SSN>
            </item>
          </records>
        </ns:DBupdate>
        <id id="_1" xsi:type="xsd:string">Joe</id>
      </SOAP-ENV:Body>
    </SOAP-ENV:Envelope>

Note that the name "Joe" is shared by two records and the string is referenced
by SOAP 1.1 href and id attributes.

While gSOAP only introduces multi-referenced elements in the payload when they
are actually multi-referenced in the data graph, other SOAP applications may
render multi-referenced elements more aggressively.  The example could also be
rendered as:

    <SOAP-ENV:Envelope
      xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
      xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:xsd="http://www.w3.org/2001/XMLSchema"
      xmlsn:ns="urn:DB">
      <SOAP-ENV:Body SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
        <ns:DBupdate>
          <records SOAP-ENC:arrayType="ns:record[3]">
            <item href="#id1"/>
            <item href="#id2"/>
            <item href="#id3"/>
          </records>
        </ns:DBupdate>
        <id id="id1" xsi:type="ns:record">
          <name href="#id4"/>
          <SSN>1234567890</SSN>
        </id>
        <id id="id2" xsi:type="ns:record">
          <name href="#id5"/>
          <SSN>1987654320</SSN>
        </id>
        <id id="id3" xsi:type="ns:record">
          <name href="#id4"/>
          <SSN>2345678901</SSN>
        </id>
        <id id="id4" xsi:type="xsd:string">Joe</id>
        <id id="id5" xsi:type="xsd:string">Jane</id>
      </SOAP-ENV:Body>
    </SOAP-ENV:Envelope>

SOAP 1.2 encoding is cleaner and produces more accurate XML encodings of data
graphs by setting the id attribute on the element that is referenced:

    <SOAP-ENV:Envelope
      xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope"
      xmlns:SOAP-ENC="http://www.w3.org/2003/05/soap-encoding"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:xsd="http://www.w3.org/2001/XMLSchema"
      xmlsn:ns="urn:DB">
      <SOAP-ENV:Body>
        <ns:DBupdate SOAP-ENV:encodingStyle="http://www.w3.org/2003/05/soap-encoding">
          <records SOAP-ENC:itemType="ns:record" SOAP-ENC:arraySize="3">
            <item>
              <name SOAP-ENC:id="_1">Joe</name>
              <SSN>1234567890</SSN>
            </item>
            <item>
              <name>Jane</name>
              <SSN>1987654320</SSN>
            </item>
            <item>
              <name SOAP-ENC:ref="_1"/>
              <SSN>2345678901</SSN>
            </item>
          </records>
        </ns:DBupdate>
      </SOAP-ENV:Body>
    </SOAP-ENV:Envelope>

@note Some SOAP 1.2 applications consider the namespace `SOAP-ENC` of
`SOAP-ENC:id` and `SOAP-ENC:ref` optional.  The gSOAP SOAP 1.2 encoding
serialization follows the 2007 standard, while accepting unqualified id and
ref attributes.

To remove all rendered id-ref multi-referenced elements in gSOAP, use the
`SOAP_XML_TREE` flag to initialize the gSOAP engine context.

Some XML validation rules are turned off with SOAP encoding, because of the
presence of additional attributes, such as id and ref/href, SOAP arrays with
arbitrary element tags for array elements, and the occurrence of additional
multi-ref elements in the SOAP 1.1 Body.

The use of "literal" puts no restrictions on the XML in the SOAP Body.  Full
XML validation is possible, which can be enabled with the `SOAP_XML_STRICT`
flag to initialize the gSOAP engine context.  However, data graphs will be
serialized as trees and cycles in the data will be cut from the XML rendition.


SOAP 1.1 versus SOAP 1.2                                                 {#soap}
------------------------

There are two SOAP protocol versions: 1.1 and 1.2. The gSOAP tools can switch
between the two versions seamlessly.  You can declare the default SOAP version
for a service operation as follows:

      //gsoap ns service method-protocol: DBupdate SOAP1.2

The gSOAP soapcpp2 auto-generates client and server code.  At the client side,
this operation sends data with SOAP 1.2 but accepts responses also in SOAP 1.1.
At the server side, this operation accepts requests in SOAP 1.1 and 1.2 and
will return responses in the same SOAP version.

As we discussed in the previous section, the SOAP 1.2 protocol has a cleaner
multi-referenced element serialization format that greatly enhances the
accuracy of data graph serialization with SOAP RPC encoding and is therefore
recommended.

The SOAP 1.2 protocol default can also be set by importing and loading
`gsoap/import/soap12.h`:

    #import "soap12.h"


Non-SOAP XML serialization                                           {#non-soap}
--------------------------

You can serialize data that is stored on the heap, on the stack (locals), and
static data as long as the serializable (i.e. non-transient) members are
properly initialized and pointers in the structures are either NULL or point to
valid structures.  Deserialized data is put on the heap and managed by the
gSOAP engine context `struct soap`, see also [Memory management](#memory).

You can read and write XML directly to a file or stream with the serializing
[Operations on classes and structs](#toxsd9-11).

To define and use XML Web service client and service operations, we can declare
these operations in our gSOAP header file for soapcpp2 as functions that
soapcpp2 will translate in client-side service invocation calls and server-side
service operation dispatchers.  These functions are auto-generated by wsdl2h
from WSDLs.  Note that XSDs do not include service definitions.

The REST operations POST, GET, and PUT are declared with gSOAP directives in
the gSOAP header file for soapcpp2.  For example, a REST POST operation is
declared as follows:

    //gsoap ns service namespace:       urn:DB
    //gsoap ns service method-protocol: DBupdate POST
    int ns__DBupdate(...);

There is no SOAP Envelope and no SOAP Body in the payload for `DBupdate`.  Also
the XML serialization rules are identical to SOAP document/literal.  The XML
payload only has the operation name as an element with its parameters
serialized as subelements:

    <ns:DBupdate xmln:ns="urn:DB" ...>
     ...
    </ns:DBupdate>

To force id-ref serialization with REST similar to SOAP 1.2 multi-reference
encoding, use the `SOAP_XML_GRAPH` flag to initialize the gSOAP engine context.
The XML serialization includes id and ref attributes for multi-referenced
elements as follows:

    <ns:DBupdate xmln:ns="urn:DB" ...>
      <records>
        <item>
          <name id="_1">Joe</name>
          <SSN>1234567890</SSN>
        </item>
        <item>
          <name>Jane</name>
          <SSN>1987654320</SSN>
        </item>
        <item>
          <name ref="_1"/>
          <SSN>2345678901</SSN>
        </item>
      </records>
    </ns:DBupdate>


Memory management                                                      {#memory}
=================

Memory management with the `soap` context enables us to allocate data in
context-managed heap space that can be collectively deleted.  All deserialized
data is placed on the context-managed heap by the gSOAP engine.


Memory management in C                                                {#memory1}
----------------------

In C (wsdl2h option `-c` and soapcpp2 option `-c`), the gSOAP engine allocates
data on a context-managed heap with:

- `void *soap_malloc(struct soap*, size_t len)`.

The `soap_malloc` function is a wrapper around `malloc`, but which also allows
the `struct soap` context to track all heap allocations for collective deletion
with `soap_end(soap)`:

    #include "soapH.h"
    #include "ns.nsmap"
    ...
    struct soap *soap = soap_new();  // new context
    ...
    struct ns__record *record = soap_malloc(soap, sizeof(struct ns__record));
    soap_default_ns__record(soap, record);
    ...
    soap_destroy(soap);  // only for C++, see section on C++ below
    soap_end(soap);      // delete record and all other heap allocations
    soap_free(soap);     // delete context

The soapcpp2 auto-generated deserializers in C use `soap_malloc` to allocate
and populate deserialized structures, which are managed by the context for
collective deletion.

To make `char*` and `wchar_t*` string copies to the context-managed heap, we
can use the functions:

- `char *soap_strdup(struct soap*, const char*)` and
- `wchar_t *soap_wstrdup(struct soap*, const wchar_t*)`.

We use the soapcpp2 auto-generated `soap_dup_T` functions to duplicate data
into another context (this requires soapcpp2 option `-Ec` to generate), here
shown for C with the second argument `dst` NULL because we want to allocate a
new managed structure:

    struct soap *other_soap = soap_new();  // another context
    struct ns__record *other_record = soap_dup_ns__record(other_soap, NULL, record);
    ...
    soap_destroy(other_soap);  // only for C++, see section on C++ below
    soap_end(other_soap);      // delete other_record and all of its deep data
    soap_free(other_soap);     // delete context

Note that the only reason to use another context and not to use the primary
context is when the primary context must be destroyed together with all of the
objects it manages while some of the objects must be kept alive.  If the objects
that are kept alive contain deep cycles then this is the only option we have,
because deep copy with a managing context detects and preserves these
cycles unless the `SOAP_XML_TREE` flag is used with the context:

    struct soap *other_soap = soap_new1(SOAP_XML_TREE);  // another context
    struct ns__record *other_record = soap_dup_ns__record(other_soap, NULL, record);

The resulting deep copy will be a full copy of the source data structure as a
tree without co-referenced data (i.e. no digraph) and without cycles.  Cycles
are pruned and (one of the) pointers that forms a cycle is repaced by NULL.

We can also deep copy into unmanaged space and use the auto-generated
`soap_del_T()` function (requires soapcpp2 option `-Ed` to generate) to delete
it later, but we MUST NOT do this for any data that we suspect has deep cycles:

    struct ns__record *other_record = soap_dup_ns__record(NULL, NULL, record);
    ...
    soap_del_ns__record(other_record);  // deep delete record data members
    free(other_record);                 // delete the record

Cycles in the data structure will lead to non-termination when making unmanaged
deep copies.  Consider for example:

    struct ns__record
    {
      const char  *name;
      uint64_t     SSN;
      ns__record  *spouse;
    };

Our code to populate a structure with a mutual spouse relationship:

    struct soap *soap = soap_new();
    ...
    struct ns__record pers1, pers2;
    soap_default_ns__record(soap, &pers1);
    soap_default_ns__record(soap, &pers2);
    pers1.name = "Joe";                     // OK to serialize static data
    pers1.SSN = 1234567890;
    pers1.spouse = &pers2;
    pers2.name = soap_strdup(soap, "Jane"); // allocates and copies a string
    pers2.SSN = 1987654320;
    pers2.spouse = &pers1;
    ...
    struct ns__record *pers3 = soap_dup_ns__record(NULL, NULL, &pers1);  // BAD
    struct ns__record *pers4 = soap_dup_ns__record(soap, NULL, &pers1);  // OK
    soap_set_mode(soap, SOAP_XML_TREE);
    struct ns__record *pers5 = soap_dup_ns__record(soap, NULL, &pers1);  // OK

As we can see, the gSOAP serializer can serialize any heap, stack, or static
allocated data, such as in our code above.  So we can serialize the
stack-allocated `pers1` record as follows:

    soap->sendfd = fopen("record.xml", "w");
    soap_set_mode(soap, SOAP_XML_GRAPH);  // support id-ref w/o requiring SOAP
    soap_clr_mode(soap, SOAP_XML_TREE);   // if set, clear
    soap_write_ns__record(soap, &pers1);
    fclose(soap->sendfd);
    soap->sendfd = NULL;

which produces an XML document record.xml that is similar to:

    <ns:record xmlns:ns="urn:types" id="Joe">
      <name>Joe</name>
      <SSN>1234567890</SSN>
      <spouse id="Jane">
        <name>Jane</name>
        <SSN>1987654320</SSN>
        <spouse ref="#Joe"/>
      </spouse>
    </ns:record>

Deserialization of an XML document with a SOAP 1.1/1.2 encoded id-ref graph
leads to the same non-termination problem when we later try to copy the data
into unmanaged space:
    
    struct soap *soap = soap_new1(SOAP_XML_GRAPH);  // support id-ref w/o SOAP
    ...
    struct ns__record pers1;
    soap->recvfd = fopen("record.xml", "r");
    soap_read_ns__record(soap, &pers1);
    fclose(soap->recvfd);
    soap->recvfd = NULL;
    ...
    struct ns__record *pers3 = soap_dup_ns__record(NULL, NULL, &pers1);  // BAD
    struct ns__record *pers4 = soap_dup_ns__record(soap, NULL, &pers1);  // OK
    soap_set_mode(soap, SOAP_XML_TREE);
    struct ns__record *pers5 = soap_dup_ns__record(soap, NULL, &pers1);  // OK

Copying data with `soap_dup_T(soap)` into managed space is always safe.  Copying
into unmanaged space requires diligence.  But deleting unmanaged data is easy
with `soap_del_T()`.

We can also use `soap_del_T()` to delete structures that we created in C, but
only if these structures are created with `malloc` and do NOT contain pointers
to stack and static data.


Memory management in C++                                              {#memory2}
------------------------

In C++, the gSOAP engine allocates data on a managed heap using a combination
of `void *soap_malloc(struct soap*, size_t len)` and `soap_new_T()`, where `T`
is the name of a class, struct, or class template (container or smart pointer).
Heap allocation is tracked by the `struct soap` context for collective
deletion with `soap_destroy(soap)` and `soap_end(soap)`.

Only structs, classes, and class templates are allocated with `new` via
`soap_new_T(struct soap*)` and mass-deleted with `soap_destroy(soap)`.

There are four variations of `soap_new_T` for class/struct/template type `T`
that soapcpp2 auto-generates to create instances on a context-managed heap:

- `T * soap_new_T(struct soap*)` returns a new instance of `T` with default data
  member initializations that are set with the soapcpp2 auto-generated `void
  T::soap_default(struct soap*)` method), but ONLY IF the soapcpp2
  auto-generated default constructor is used that invokes `soap_default()` and
  was not replaced by a user-defined default constructor.

- `T * soap_new_T(struct soap*, int n)` returns an array of `n` new instances of
  `T`.  Similar to the above, instances are initialized.

- `T * soap_new_req_T(struct soap*, ...)` returns a new instance of `T` and sets
  the required data members to the values specified in `...`.  The required data
  members are those with nonzero minOccurs, see the subsections on
  [(Smart) pointer members and their occurrence constraints](#toxsd9-6) and
  [Container members and their occurrence constraints](#toxsd9-7).

- `T * soap_new_set_T(struct soap*, ...)` returns a new instance of `T` and sets
  the public/serializable data members to the values specified in `...`.

The above functions can be invoked with a NULL `soap` context, but we will be
responsible to use `delete T` to remove this instance from the unmanaged heap.

Primitive types and arrays of these are allocated with `soap_malloc` by the
gSOAP engine.  As we stated above, all types except for classes, structs, class
templates (containers and smart pointers) are allocated with `soap_malloc` for
reasons of efficiency.

We can use a C++ template to simplify the managed allocation and initialization
of primitive values as follows (this is for primitive types only, because we
should allocate structs and classes with `soap_new_T`):

    template<class T>
    T * soap_make(struct soap *soap, T val)
    {
      T *p = (T*)soap_malloc(soap, sizeof(T));
      if (p)      // out of memory? Can also guard with assert(p != NULL) or throw an error
        *p = val;
      return p;
    }

For example, assuming we have the following class:

    class ns__record
    {
     public:
      std::string  name;    // required name
      uint64_t    *SSN;     // optional SSN
      ns__record  *spouse;  // optional spouse
    };

We can instantiate a record by using the auto-generated
`soap_new_set_ns__record` and our `soap_make` to create a SSN value on the
managed heap:

    soap *soap = soap_new();  // new context
    ...
    ns__record *record = soap_new_set_ns__record(
        soap,
        "Joe",
        soap_make<uint64_t>(soap, 1234567890LL),
        NULL);
    ...
    soap_destroy(soap);  // delete record and all other managed instances
    soap_end(soap);      // delete managed soap_malloc'ed heap data
    soap_free(soap);     // delete context

Note however that the gSOAP serializer can serialize any heap, stack, or static
allocated data.  So we can also create a new record as follows:

    uint64_t SSN = 1234567890LL;
    ns__record *record = soap_new_set_ns__record(soap, "Joe", &SSN, NULL);

which will be fine to serialize this record as long as the local `SSN`
stack-allocated value remains in scope when invoking the serializer and/or
using `record`.  It does not matter if `soap_destroy` and `soap_end` are called
beyond the scope of `SSN`.

To facilitate our class methods to access the managing context, we can add a
soap context pointer to a class/struct:

    class ns__record
    {
      ...
      void create_more();  // needs a context to create more internal data
     protected:
      struct soap *soap;   // the context that manages this instance, or NULL
    };

The context is set when invoking `soap_new_T` (and similar) with a non-NULL
context argument.

We use the soapcpp2 auto-generated `soap_dup_T` functions to duplicate data
into another context (this requires soapcpp2 option `-Ec` to generate), here
shown for C++ with the second argument `dst` NULL because we want to allocate a
new managed object:

    soap *other_soap = soap_new();  // another context
    ns__record *other_record = soap_dup_ns__record(other_soap, NULL, record);
    ...
    soap_destroy(other_soap);  // delete record and other managed instances
    soap_end(other_soap);      // delete other data (the SSNs on the heap)
    soap_free(other_soap);     // delete context

To duplicate base and derived instances when a base class pointer or reference
is provided, use the auto-generated method `T * T::soap_dup(struct soap*)`:

    soap *other_soap = soap_new();  // another context
    ns__record *other_record = record->soap_dup(other_soap);
    ...
    soap_destroy(other_soap);  // delete record and other managed instances
    soap_end(other_soap);      // delete other data (the SSNs on the heap)
    soap_free(other_soap);     // delete context

Note that the only reason to use another context and not to use the primary
context is when the primary context must be destroyed together with all of the
objects it manages while some of the objects must be kept alive.  If the objects
that are kept alive contain deep cycles then this is the only option we have,
because deep copy with a managing context detects and preserves these
cycles unless the `SOAP_XML_TREE` flag is used with the context:

    soap *other_soap = soap_new1(SOAP_XML_TREE);  // another context
    ns__record *other_record = record->soap_dup(other_soap);  // deep tree copy

The resulting deep copy will be a full copy of the source data structure as a
tree without co-referenced data (i.e. no digraph) and without cycles.  Cycles
are pruned and (one of the) pointers that forms a cycle is repaced by NULL.

We can also deep copy into unmanaged space and use the auto-generated
`soap_del_T()` function or the `T::soap_del()` method (requires soapcpp2 option
`-Ed` to generate) to delete it later, but we MUST NOT do this for any data
that we suspect has deep cycles:

    ns__record *other_record = record->soap_dup(NULL);
    ...
    other_record->soap_del();  // deep delete record data members
    delete other_record;       // delete the record

Cycles in the data structure will lead to non-termination when making unmanaged
deep copies.  Consider for example:

    class ns__record
    {
      const char  *name;
      uint64_t     SSN;
      ns__record  *spouse;
    };

Our code to populate a structure with a mutual spouse relationship:

    soap *soap = soap_new();
    ...
    ns__record pers1, pers2;
    pers1.name = "Joe";
    pers1.SSN = 1234567890;
    pers1.spouse = &pers2;
    pers2.name = "Jane";
    pers2.SSN = 1987654320;
    pers2.spouse = &pers1;
    ...
    ns__record *pers3 = soap_dup_ns__record(NULL, NULL, &pers1);  // BAD
    ns__record *pers4 = soap_dup_ns__record(soap, NULL, &pers1);  // OK
    soap_set_mode(soap, SOAP_XML_TREE);
    ns__record *pers5 = soap_dup_ns__record(soap, NULL, &pers1);  // OK

Note that the gSOAP serializer can serialize any heap, stack, or static
allocated data, such as in our code above.  So we can serialize the
stack-allocated `pers1` record as follows:

    soap->sendfd = fopen("record.xml", "w");
    soap_set_mode(soap, SOAP_XML_GRAPH);  // support id-ref w/o requiring SOAP
    soap_clr_mode(soap, SOAP_XML_TREE);   // if set, clear
    soap_write_ns__record(soap, &pers1);
    fclose(soap->sendfd);
    soap->sendfd = NULL;

which produces an XML document record.xml that is similar to:

    <ns:record xmlns:ns="urn:types" id="Joe">
      <name>Joe</name>
      <SSN>1234567890</SSN>
      <spouse id="Jane">
        <name>Jane</name>
        <SSN>1987654320</SSN>
        <spouse ref="#Joe"/>
      </spouse>
    </ns:record>

Deserialization of an XML document with a SOAP 1.1/1.2 encoded id-ref graph
leads to the same non-termination problem when we later try to copy the data
into unmanaged space:
    
    soap *soap = soap_new1(SOAP_XML_GRAPH);  // support id-ref w/o SOAP
    ...
    ns__record pers1;
    soap->recvfd = fopen("record.xml", "r");
    soap_read_ns__record(soap, &pers1);
    fclose(soap->recvfd);
    soap->recvfd = NULL;
    ...
    ns__record *pers3 = soap_dup_ns__record(NULL, NULL, &pers1);  // BAD
    ns__record *pers4 = soap_dup_ns__record(soap, NULL, &pers1);  // OK
    soap_set_mode(soap, SOAP_XML_TREE);
    ns__record *pers5 = soap_dup_ns__record(soap, NULL, &pers1);  // OK

Copying data with `soap_dup_T(soap)` into managed space is always safe.  Copying
into unmanaged space requires diligence.  But deleting unmanaged data is easy
with `soap_del_T()`.

We can also use `soap_del_T()` to delete structures in C++, but only if these
structures are created with `new` (and `new []` for arrays when applicable) for
classes, structs, and class templates and with `malloc` for anything else, and
the structures do NOT contain pointers to stack and static data.

Features and limitations                                             {#features}
========================

In general, to use the generated code:

- Make sure to `#include "soapH.h"` in your code and also define a namespace
  table or `#include "ns.nsmap"` with the generated table, where `ns` is the
  namespace prefix for services.

- Use soapcpp2 option -j (C++ only) to generate C++ proxy and service objects.
  The auto-generated files include documented inferfaces.  Compile with
  soapC.cpp and link with -lgsoap++, or alternatively compile stdsoap2.cpp.

- Without soapcpp2 option -j: client-side uses the auto-generated
  soapClient.cpp and soapC.cpp (or C versions of those).  Compile and link with
  -lgsoap++ (-lgsoap for C), or alternatively compile stdsoap2.cpp
  (stdsoap2.c for C).

- Without soapcpp2 option -j: server-side uses the auto-generated
  soapServer.cpp and soapC.cpp (or C versions of those).  Compile and link with
  -lgsoap++ (-lgsoap for C), or alternatively compile stdsoap2.cpp (stdsoap2.c
  for C).

- Use `soap_new()` or `soap_new1(int flags)` to allocate and initialize a
  heap-allocated context with or without flags.  Delete this context with
  `soap_free(struct soap*)`, but only after `soap_destroy(struct soap*)` and
  `soap_end(struct soap*)`.

- Use `soap_init(struct *soap)` or `soap_init1(struct soap*, int flags)` to
  initialize a stack-allocated context with or without flags.  End the use of
  this context with `soap_done(struct soap*)`, but only after
  `soap_destroy(struct soap*)` and `soap_end(struct soap*)`.

There are several context initialization flags and context mode flags to
control XML serialization at runtime:

- `SOAP_C_UTFSTRING`: enables all `std::string` and `char*` strings to
  contain UTF-8 content.  This option is recommended.

- `SOAP_XML_STRICT`: strictly validates XML while deserializing.  Should not be
  used together with SOAP 1.1/1.2 encoding style of messaging.  Use soapcpp2
  option `-s` to hard code `SOAP_XML_STRICT` in the generated serializers.  Not
  recommended with SOAP 1.1/1.2 encoding style messaging.

- `SOAP_XML_INDENT`: produces indented XML.

- `SOAP_XML_CANONICAL`: c14n canonocalization, removes unused `xmlns` bindings
  and adds them to appropriate places by applying c14n normalization rules.
  Should not be used together with SOAP 1.1/1.2 encoding style messaging.

- `SOAP_XML_TREE`: write tree XML without id-ref, while pruning data structure
  cycles to prevent nontermination of the serializer for cyclic structures.

- `SOAP_XML_GRAPH`: write graph (digraph and cyclic graphs with shared pointers
  to objects) using id-ref attributes.  That is, XML with SOAP multi-ref
  encoded id-ref elements.  This is a structure-preserving serialization format,
  because co-referenced data and also cyclic relations are accurately represented.

- `SOAP_XML_DEFAULTNS`: uses xmlns default bindings, assuming that the schema
  element form is "qualified" by default (be warned if it is not!).

- `SOAP_XML_NOTYPE`: removes all xsi:type attribuation.  This option is usually
  not needed unless the receiver rejects all xs:type attributes.  This option
  may affect the quality of the deserializer, which relies on xsi:type
  attributes to distinguish base class instances from derived class instances
  transported in the XML payloads.

Additional notes with respect to the wsdl2h and soapcpp2 tools:

- Nested classes, structs, and unions in a gSOAP header file are unnested by
  soapcpp2.

- Use `#import "file.h"` instead of `#include` to import other header files in
  a gSOAP header file for soapcpp2.  The `#include` and `#define` directives are
  accepted, but are moved to the very start of the generated code for the C/C++
  compiler to include before all generated definitions.  You should use
  `#include` in combinatio with "volatile" types and to ensure transient
  (incomplete) types are declared when these are used in the gSOAP header file.

- To remove any SOAP-specific bindings, use soapcpp2 option `-0`.

- A gSOAP header file for soapcpp2 should not include any code statements, only
  data type declarations.  This includes constructor initialization lists that are
  not permitted.  Use member initializations instead.

- C++ namespaces are supported.  Use wsdl2h option `-qname`.  Or add a `namespace
  name { ... }` to the header file, but the `{ ... }` MUST cover the entire
  header file content from begin to end.
  
- Optional DOM support can be used to store mixed content or literal XML
  content.  Otherwise, mixed content may be lost.  Use wsdl2h option `-d` for
  DOM support and compile and link with `dom.c` or `dom.cpp`.


Removing SOAP namespaces from XML payloads                              {#nsmap}
==========================================

The soapcpp2 tool generates a `.nsmap` file that includes two bindings for SOAP
namespaces.  We can remove all SOAP namespaces (and SOAP processing logic) with
soapcpp2 option `-0` or by simply setting the two entries to NULL:

    struct Namespace namespaces[] =
    {
      {"SOAP-ENV", NULL, NULL, NULL},
      {"SOAP-ENC", NULL, NULL, NULL},
      ...

Note that once the `.nsmap` is generated, we can copy-paste the content into
our project code.  However, if we rerun wsdl2h on updated WSDL/XSD files or
`typemap.dat` declarations then we need to use the updated table.

In cases that no XML namespaces are used at all, for example with
[XML-RPC](www.genivia.com/doc/xml-rpc-json/html), you may use an empty
namespace table:

    struct Namespace namespaces[] = {{NULL,NULL,NULL,NULL}};

However, beware that any built-in xsi attributes that are rendered will lack
the proper namespace binding.  At least we suggest to use `SOAP_XML_NOTYPE` for
this reason.

Examples                                                             {#examples}
========

Select the project files below to peruse the source code examples.


Source files
------------

- `address.xsd`         Address book schema
- `address.cpp`         Address book app (reads/writes address.xml file)
- `addresstypemap.dat`  Schema namespace prefix name preference for wsdl2h
- `graph.h`             Graph data binding (tree, digraph, cyclic graph)
- `graph.cpp`           Test graph serialization as tree, digraph, and cyclic


Generated files
---------------

- `address.h`      gSOAP-specific data binding definitions from address.xsd
- `addressStub.h`  C++ data binding definitions
- `addressH.h`     Serializers
- `addressC.cpp`   Serializers
- `address.xml`    Address book data generated by address app
- `graphStub.h`    C++ data binding definitions
- `graphH.h`       Serializers
- `graphC.cpp`     Serializers
- `g.xsd`          XSD schema with `g:Graph` complexType
- `g.nsmap`        xmlns bindings namespace mapping table


Build steps
-----------

Building the AddressBook example:

    wsdl2h -g -t addresstypemap.dat address.xsd
    soapcpp2 -0 -CS -I../../import -p address address.h
    c++ -I../.. address.cpp addressC.cpp -o address -lgsoap++

Option `-g` produces bindings for global (root) elements in addition to types.
In this case the root element `a:address-book` is bound to `_a__address_book`.
The complexType `a:address` is bound to class `a__address`, which is also the
type of `_a__address_book`.  This option is not required, but allows you to use
global element tag names when referring to their serializers, instead of their
type name.  Option `-0` removes the SOAP protocol.  Options `-C` and `-S`
removes client and server code generation.  Option `-p` renames the output
`soap` files to `address` files.

See the `address.cpp` implementation and [Related Pages](pages.html).

The `addresstypemap.dat` file specifies the XML namespace prefix for the
bindings:

    #       Bind the address book schema namespace to prefix 'a'

    a = "urn:address-book-example"

    #       By default the xsd:dateTime schema type is translated to time_t
    #       To map xsd:dateTime to struct tm, enable the following line:

    # xsd__dateTime = #import "../../custom/struct_tm.h"

    #       ... and compile/link with custom/struct_tm.c

The DOB field is a xsd:dateTime, which is bound to `time_t` by default.  To
change this to `struct tm`, enable the import of the `xsd__dateTime` custom
serializer by uncommenting the definition of `xsd__dateTime` in
`addresstypemap.dat`.  Then change `soap_dateTime2s` to `soap_xsd__dateTime2s`
in the code.

Building the graph serialization example:

    soapcpp2 -CS -I../../import -p graph graph.h
    c++ -I../.. graph.cpp graphC.cpp -o graph -lgsoap++

To compile without using the `libgsoap++` library: simply compile
`stdsoap2.cpp` together with the above.


Usage
-----

To execute the AddressBook example:

    ./address

To execute the Graph serialization example:

    ./graph

