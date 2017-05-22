# A Mapping from JSON to Linked Data

_This is an initial draft specification , and has been produced to provide an easier to  understand serialisation for web of thing interactions models — Dave Raggett_

The Internet of Things suffers from fragmentation with many platforms and barriers to integration. The Web of Things provides an interoperability framework that aims to reduce the effort and risk for providing  services and enable open markets of services across a broad range of IoT platforms.

The Web of Things deals with local or remote, physical or abstract, things that applications can either provide or consume as software objects. Applications are decoupled from the underlying protocols, data formats, as well as their encodings, and communication patterns. This makes it easy to provide and consume services across a wide range of platforms and standards.

Each thing in the Web of Things requires a description of its interaction model. This must be accessible via a Uniform Resource Identifier (URI), see [RFC3986](https://www.ietf.org/rfc/rfc3986.txt). The interaction model provides a [programming language independent abstract description](td-draft-spec-dsr.md) of how applications can interact with things in terms of their properties, actions, events and metadata.

Interaction models can be declared in JSON as an alternative to other serialisations of Linked Data. This makes the models easier to understand than other serializations, including [JSON-LD](https://www.w3.org/TR/json-ld/), particularly so for more complex models. Here is an example of an interaction model expressed first as Turtle and then as JSON:

```Turtle
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix td: <http://www.w3.org/ns/td#> .
@prefix ex: <http://example.com/vocab/#> .

<http://example.com/water> a td:thing ;
	td:property _:1 , _:2 , _:3 , _:4 , _:5 ;
	td:event _:6 , _:7 ;
	td:context <http://example.com/context.json> .
_:1 td:name ex:inlet ;
	td:type td:boolean .
_:2 td:name ex:outlet ;
	td:type td:boolean .
_:3 td:name "upper" ;
	td:type td:number .
_:4 td:name "lower" ;
	td:type td:number .
_:5 td:name ex:level ;
	td:type td:number .
_:6 td:name "high_water" .
_:7 td:name "low_water" . 
```

and

```JSON
{
    "@context": "context.json",
    "events": {
        "high_water": null,
        "low_water": null
    },
    "properties": {
        "inlet": "boolean",
        "outlet": "boolean",
        "upper": "number",
        "lower": "number",
        "level": "number"
    }
}
```

The JSON object property names "@context",  "@prefix" and "@base" are reserved.  The value of "@context" is a URI that links to a file consisting of a single JSON object that maps names to RDF node URIs. "@context" is likewise reserved for links to further contexts. "@context" is used to translate JSON object property names representing RDF predicates to URIs. Context files may use "@prefix" to bind namespace prefixes to a string, that is then used to expand prefixed names to URIs using the same algorithm as [Turtle](https://www.w3.org/TR/turtle/). The value for "@prefix" must be a JSON object whose property names are interpreted as prefix declarations, and the values as the URI for the namespace. "@base" may be used to define a base URI for expanding relative URIs.

Nested JSON objects are interpreted as either defining predicates, or defining names, alternating between them at each successive level of nesting. In the following example properties, type, units and writeable are interpreted as predicates, whilst acceleration is interpreted as a name.

```JSON
{
    "@context": "context.json",
    "properties": {
        "acceleration": {
            "type": "number",
            "units": "g",
            "writeable": false
        }
    }
}
```

The default context is assumed to contain the following mappings 

* "properties" to _td:property_
* "actions" to _td:action_
* "events" to _td:event_
* "types" to _td:typedef_

The default context could thus take the form:

```JSON
{
    "@prefix" : {
        "td" : "<http://www.w3.org/ns/td#>",
    },
    "properties" : "td:property",
    "actions" : "td:action",
    "events" : "td:event"
}
```

"@context" may be used in the interaction model at any level where the JSON object properties will be interpreted as predicates.

For each JSON object property that is interpreted as a name, an RDF node is constructed, e.g. as a new blank node. The name is then declared as an RDF string literal for a triple with the predicate _td:name_. The node then becomes the subject for all predicates defined by a JSON object that is the value of the name.

When JSON object properties are interpreted as predicates, and their values are string literals, these values are mapped to URIs via the context. You can disable this mapping by declaring the string as null in an inner context. In general, contexts are searched for mappings starting with the current JSON object and ascending the hierarchy of nested JSON objects until the default context has been searched. If no mapping is found, the string value is translated as is. If a mapping cannot be found for a predicate, this constitutes an error. 

In addition, the URI for the interaction model is used to declare that the given thing is an instance of the class _td:thing_. The prefix declarations in the context are used to generate the corresponding declarations when transforming to Turtle.

Applying this algorithm to the previous example, we get:

```Turtle
@prefix td: <http://www.w3.org/ns/td#> .

<http://example/accelerometer> a td:thing ;
	td:property _:1 .
_:1 td:name "acceleration" ;
	td:type td:number ;
	td:units "g" ;
	td:writeable false . 
```

If the value of a JSON object property acting as name is a JSON array, the array items are interpreted as defining an enumeration. For example:

```JSON
{
    "properties": {
        "color": [
            "red",
            "green",
            "blue"
        ]
    }
}
```

which translates to:

```Turtle
@prefix td: <http://www.w3.org/ns/td#> .

<http://example.com/color> a td:thing ;
	td:property _:1 .
_:1 td:name "color" ;
	td:type td:enum ;
	td:item "red" , "green" , "blue" . 
```

If the value of a JSON object property acting as a name is a string, this is interpreted as a type, and mapped to a predicate using the context. For example:

```JSON
{
    "properties" : {
        "switch" : "boolean"
    }
}
```

is translated to:

```Turtle
@prefix td: <http://www.w3.org/ns/td#> .

<http://example.com/switch> a td:thing ;
	td:property _:1 .
_:1 td:name "switch" ;
	td:type td:boolean . 
```

If the string cannot be mapped to an RDF node, this constitutes an error. It is also an error if the value of a JSON object property acting as name is neither a string nor an object.

If the name of a JSON object property acting as a predicate is "types", then its value must be a JSON object whose property names are interpreted as the names of application defined  types. The property value is interpreted in the regular way. The type name can then be used in the same way as the names for core types.

Here is an example that defines a type named acceleration and use it for each axis of an accelerometer:

```JSON
{
    "types": {
        "acceleration": {
            "type": "number",
            "min": -15,
            "max": 15,
            "rate": 32
        }
    },
    "properties": {
        "value": {
            "x": "acceleration",
            "y": "acceleration",
            "z": "acceleration"
        }
    }
}
```

which translates to:

```Turtle
@prefix td: <http://www.w3.org/ns/td#> .

<http://example.com/acceleration> a td:thing ;
	td:typedef _:1 ;
	td:property _:2 .
_:1 td:name "acceleration" ;
	td:type td:number ;
	td:min -15 ;
	td:max 15 ;
	td:rate 32 .
_:2 td:name "value" ;
	td:property _:3 , _:4 , _:5 .
_:3 td:name "x" ;
	td:type _:1 .
_:4 td:name "y" ;
	td:type _:1 .
_:5 td:name "z" ;
	td:type _:1 . 
```

Note that this example could be rewritten using a Vector, e.g.

```JSON
{
    "types": {
        "acceleration": {
            "type": "number",
            "min": -15,
            "max": 15,
            "rate": 32
        }
    },
    "properties": {
        "value": {
            "vector": [
            	"x", "y", "z"
            ],
	    "type": "acceleration"
        }
    }
}
```

which translates to:

```Turtle
@prefix td: <http://www.w3.org/ns/td#> .

<http://example.com/acceleration> a td:thing ;
	td:typedef _:1 ;
	td:property _:2 .
_:1 td:name "acceleration" ;
	td:type td:number ;
	td:min -15 ;
	td:max 15 ;
	td:rate 32 .
_:2 td:name "value" ;
	td:type td:vector ;
    td:item _:3 , _:4 , _:5 ;
    td:itemType _:1 .
_:3 td:order 0 ;
	td:name "x" .
_:4 td:order 2 ;
	td:name "y" .
_:5 ud:order 3 ;
	td:name "z" . 
```

This approach has been applied to a wide range of examples, see:

* https://www.w3.org/WoT/demos/td2ttl/
* https://www.w3.org/WoT/demos/td2ttl/oic.html
* https://www.w3.org/WoT/demos/td2ttl/m2m.html