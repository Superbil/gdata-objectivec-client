# Google Data APIs Objective-C Client Library Design Document #

## Document Audience ##

The audience for this document is developers who will be maintaining
or extending the Google Data APIs Objective-C Client Library.

Readers should already be familiar with [Developer Introduction to the Google Data APIs Objective-C Client Library](GDataObjCIntroduction.md).

The final section of this document examines the implementation of a single GData element class, and may be a good starting point for readers who learn best from examples.

## Project Goals ##

The Google Data APIs Objective-C Client Library is intended to make it easy for Cocoa applications to interact with Google services.

The library handles parsing and generation of XML, HTTP network transactions, and authentication (signing in to Google accounts).

The library does not provide any user interface (aside from optional !OAuth sign-in), but the object interfaces should work smoothly with KVC and Cocoa bindings.

On the Mac, the library relies on AppKit's [NSXMLDocument](http://developer.apple.com/documentation/Cocoa/Conceptual/NSXML_Concepts/) for parsing and generating XML text. Internally, the framework builds GData objects from trees of NSXMLNodes, and generates trees of NSXMLNodes.

On the iPhone, the library uses a wrapper, GDataXML, on top of libxml2, to simulate availability of NSXMLDocument and NSXMLNode.

## Framework Objects ##

The framework defines these kinds of objects:

  * Elements, which can be created by or generate XML.  Elements include
    * Entries, derived from GDataEntryBase
    * Feeds, derived from GDataFeedBase, containing lists of entries
    * Individual data types, which may optionally be usable as extensions

  * Data-type helpers, such as GDataDateTime

  * Services, which handle sign-in and networking
    * Service objects maintain cookies, cache data, and preserve ClientLogin authentication state
    * A service ticket is issued by a service object to the client for each separate fetch request

  * Queries, which provide a convenient way to create request URLs

## GDataObjects ##

Objects in the framework which parse or generate XML are derived from the base class GDataObject.  Each subclass of GDataObject should implement these methods:

```
- (id)initWithXMLElement:(NSXMLElement *)element
                  parent:(GDataObject *)parent; // parses the XML
- (NSXMLElement *)XMLElement; // generates the XML; should begin by calling GDataObject's XMLElementWithExtensionsAndDefaultName
```
along with these usual NSObject methods:
```
- (id)copyWithZone:(NSZone *)zone; (be sure to call superclass)
- (BOOL)isEqual:(GDataObject *)other; (be sure to call superclass)
- (NSString *)description; // entries and feeds may implement -itemsForDescription instead
```

GDataObjects contain no XML data (that is, they retain no XMLNodes from the parse tree) _except_ for any unknown elements and attributes.  Unknown elements and attributes are the NSXMLNodes which were not recognized during GData parsing of the XML tree.

## Contents of the GDataObject base class ##

The GDataObject base class manages many kinds of data for each instance.  To preserve this data, derived classes _must_ call into the base class for XML parsing and generation, and to copy or compare instances of the class.

Each GDataObject may have these properties:

  * Element name
> The element's qualified name is saved from the original parsed XML, and reused during XML generation.  Extensions also declare a default element name for use in objects which were not initialized from XML.

  * Parent
> The parent object in the GData hierarchy is used only when looking for appropriate extension class declarations during parsing.  It may be nil.

  * Namespaces
> Namespaces are stored as an NSDictionary of mappings from prefix (like "gd") to URI (like "http://schemas.google.com/g/2005").  At a minimum, the root GDataObject element must declare appropriate namespaces for its XMLElement method to generate an XML tree.

  * Extension declarations
> Each GData object can specify a list of allowable extensions for itself and its descendant elements.  See "The Extension Model" below for more detail.

  * Extension instances
> Operations on extension instances (such as extension creation during XML parsing, XML generation, copying, and comparison) are handled by GDataObject for subclasses, so long as the subclasses call the superclass methods.

  * Unknown elements and unknown attributes
> At the outset of parsing the XML tree, all of an element's child elements and attributes are added to lists of unknown XMLNodes.  The XMLNodes are removed from the lists of unknowns as they are parsed.  Any remaining NSXMLNodes remaining in the lists of unknowns are retained, and are re-added during XML generation.

  * User data and properties
> A field for user data (an Objective-C `id`) and a mutable dictionary of user properties are available for client use.  It is retained by the GDataObject but otherwise ignored.

The GDataObject base class includes a wide variety of methods for these tasks:
  * Typical NSObject methods (copyWithZone, isEqual, setters, getters, and so on)
  * XML tree parsing
  * XML tree generation
  * Extension element management
  * Dynamic object class identification (see "Dynamic Object Generation" below)

## The Extension Model ##
Extensions enable an element class to parse and generate attributes and children about which the element
may know no details.

Typically, entries add extensions to themselves. For example, during initialization, a calendar
entry declares it may contain a color:
```
[self addExtensionDeclarationForParentClass:[GDataEntryCalendar class]
                                 childClass:[GDataColorProperty class]];
```
This lets the base class handle much of the work of managing the child
element.  The calendar entry can still provide accessor methods for the extension by calling into the base class, as shown here:
```
- (GDataColorProperty *)color {
  return [self objectForExtensionClass:[GDataColorProperty class]];
}

- (void)setColor:(GDataColorProperty *)val {
  [self setObject:val forExtensionClass:[GDataColorProperty class]];
}
```
Normally, elements that declare extensions for themselves will also have accessors for those extensions, such as the `color:` and `setColor:` methods shown here.

The motivating purpose of extensions is to allow elements to parse and generate attributes and children
they may not know about.  For example, links contained within calendar event entries may contain webContent elements, like this:
```
<link rel="http://schemas.google.com/gCal/2005/webContent" title="World Cup" 
      href="http://www.google.com/calendar/images/google-holiday.gif" type="image/gif"> 
  <gCal:webContent width="276" height="120" url="http://www.google.com/logos/worldcup06.gif" />  
</link>
```

To support this extension to the link element, a calendar event entry declares that GDataLinks contained within the calendar event entry may contain GDataWebContent elements:
```
[self addExtensionDeclarationForParentClass:[GDataLink class]
                                 childClass:[GDataWebContent class]];  
```
The calendar event entry has extended GDataLinks without GDataLinks knowing or
caring.  Because GDataLink derives from GDataObject, the GDataLink
object will automatically parse and maintain and copy and compare
any GDataWebContents contained within.

For extensions declared for descendant objects, Objective-C categories are useful for providing relevant accessors.  The extension declared above, where GDataLinks may contain GDataWebContents, is unknown to GDataLinks themselves.  The calendar event entry thus provides an Objective-C category on GDataLink as the accessor for the extension:
```
@implementation GDataLink (GDataCalendarEntryEventExtensions)
- (NSArray *)webContents {
  return [self objectsForExtensionClass:[GDataWebContent class]];
}

- (void)setWebContents:(NSArray *)arr {
  [self setObjects:arr forExtensionClass:[GDataWebContent class]];
}

- (void)addWebContent:(GDataWebContent *)obj {
  [self addObject:obj forExtensionClass:[GDataWebContent class]];
}
@end
```

The GDataObject base class stores the parent for each object to enable it to search during parsing for extensions declared higher up the object tree. Entries often declare extensions to the various classes of elements that may be contained in the entries.

## Dynamic Object Generation ##

Parsing XML requires knowing what kind of objects to create for that XML.  Typically, the class of a feed object will be known by the service object fetching it.  For example, a calendar app fetches a calendar by specifying the class of the feed, GDataFeedCalendar:

```
  return [self fetchFeedWithURL:feedURL 
                      feedClass:[GDataFeedCalendar class]
                       delegate:delegate
              didFinishSelector:finishedSelector];
```

Similarly, the feed class will specify the class of the entries to be created for it. Naturally, the calendar feed class expects calendar entries:

```
- (Class)classForEntries {
  return [GDataEntryCalendar class];
}
```

Sometimes, XML parsing is done without knowing in advance what the class of the XML object should be.  This happens when a service method may be used to retrieve multiple classes of objects, or when an XML feed or entry is embedded inside of an element.

To identify dynamically what class to use to parse XML when no class is specified, GDataObject uses [XPath](http://developer.apple.com/documentation/Cocoa/Conceptual/NSXML_Concepts/Articles/QueryingXML.html) to inspect `gd:kind` attributes and [category child elements](http://code.google.com/apis/gdata/elements.html#Introduction) in the XML of feeds and entries. Feed and entry classes may register in their
+[load](http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSObject_Class/Reference/Reference.html#//apple_ref/occ/clm/NSObject/load) methods the `gd:kind` or category attribute values identifying XML which can be parsed by the class. A ~YouTube video entry registers itself by category term attribute this way:
```
+ (NSString *)standardEntryKind {
  return kGDataCategoryYouTubeVideo;
}

+ (void)load {
  [self registerEntryClass];
}
```
to match the cell's category element,
```
<category scheme="http://schemas.google.com/g/2005#kind" 
          term="http://gdata.youtube.com/schemas/2007#video"/>
```

A calendar entry registers itself by `gd:kind` attribute value this way:
```
+ (NSString *)standardKindAttributeValue {
  return @"calendar#calendar";
}

+ (void)load {
  [self registerEntryClass];
}
```
to match an entry beginning like
```
<entry gd:kind="calendar#calendar">
```

## Service Classes ##

Service classes handle network transactions and [user authentication](http://code.google.com/apis/gdata/auth.html).

From the application's perspective, the model should be simple:
  1. Initiate a fetch via a URL or a query, receive a service ticket.
  1. Wait for a callback with a GData object or an error, or else use the ticket to cancel the fetch.

The base class, GDataServiceBase, does simple fetches to a URL, retrieves data, and if successful, constructs a GDataObject from the data. Authentication may be done with an HTTP credential, but no other sign-in steps are provided by the base class. A fetch and response sequence is called a "ticket."

Authentication to Google services via the ClientLogin (username/password) protocol is implemented by the GDataServiceGoogle subclass.  An authenticated fetch handled by GDataServiceGoogle can be one of several kinds of sequences of HTTP transactions. For example, a ticket for an authenticated fetch may include:

  * Authenticate the username and password with the auth server, get an auth token
  * Pass the auth token with the GData request to the service's server, get XML for a GDataObject

Or, a ticket may instead be this sequence:

  * Re-use a previous auth token in a request to the service, get a token expired error
  * Re-authenticate the username and password, get a new auth token
  * Pass the new auth token to the service's server, get XML for a GDataObject

A GDataServiceTicket object retains pointers to the service that issued it, and to the GDataHTTPFetcher objects that carry out authentication and GData requests.

The service object _does not_ maintain pointers to GDataHTTPFetchers or tickets; those are just retained by the underlying NSURLConnections while they are pending (and may also be retained indirectly by the application, if it retains the ticket.)

A service object _does_ retain references to data that applies to a series of tickets:
  * username and password
  * a fetch history dictionary, which holds cookies, last-modified dates, and caches of received, dated data for the GDataHTTPFetcher (see "Fetcher" below)
  * a userAgent string to identify the requesting application, mostly for server-side logging
  * an optional application-supplied userData value, used to initialize the userData property of future tickets

Because requests to the GData server may need to be reissued if a pre-existing token has become invalid, all parameters to an authenticated GData request are packaged up as a "retry invocation" and passed along for the sequence of HTTP transactions (except on a second request to the GData service, at which point no further retries are possible.)

The base service class GDataServiceBase simply makes a request to a GData server, creates a GData object with the returned data, and calls back to the application.

The authentication subclass, GDataServiceGoogle, makes ClientLogin requests to the authentication server, adds the authentication token to the requests created by the base class, and intercepts errors returns to the base class to handle invalid token errors.

GData service-specific subclasses add type-appropriate interfaces for the individual services, but are otherwise shallow wrappers around GDataServiceGoogle.

## Query Classes ##

GDataQuery is a simple class that maintains a list of key-value pairs, and generates a URL with the pairs appended as parameters. It also allows applications to specify category filters for queries.  GData service-specific query subclasses provide convenient accessors for the query parameters unique to the individual services.

## Fetcher ##

The utility class GDataHTTPFetcher is a wrapper around AppKit's standard HTTP transaction class, [NSURLConnection](http://developer.apple.com/documentation/Cocoa/Conceptual/URLLoadingSystem/index.html).  Like NSURLConnection, GDataHTTPFetcher does one-shot http retrievals; instances cannot be reused for additional fetches.

GDataHTTPFetcher accumulates response data for the client application, and allows each fetch to specify unique finished and failure selectors (so fetches initiated by different methods in the same object can have unique callback methods as well.)

If the user of GDataHTTPFetcher provides a mutable dictionary as fetchHistory, then the dictionary is used for maintaining state across fetches, including
  * cookies, to avoid sharing or interacting with Safari's cookies
  * "Last-Modified" header values
  * caches of data that was accompanied by "Last-Modified" headers.

GDataHTTPFetcher considers server response status of 300 and greater to be errors, and calls the client application's failure callback.

## Coding and nomenclature conventions ##

### Coding conventions ###

In general, Cocoa naming and style conventions are followed.  Formatting should be consistent, and lines should be under 80 columns wherever practical.

ivars in the framework follow the Google convention of a trailing underscore (like `NSMutableArray *entries_;`). ivars in non-framework code, such as the sample applications, have a leading character m (like `GDataFeedSpreadsheet *mSpreadsheetFeed;`).

### Class names ###

Entry, feed, service, and query classes are named consistently, with derived classes having increasingly specific names. For example, a service class inheritance tree looks like
```
GDataServiceBase
GDataServiceGoogle
GDataServiceGoogleCalendar
```
and an entry class inheritance tree looks like
```
GDataEntryBase
GDataEntryEvent
GDataEntryCalendarEvent
```
### Using "Google" ###

Within the framework, the name fragment "Google" is used in these circumstances only:
  * Service classes which authenticate the user for Google services
  * Google Base-specific classes

### Using "Base" ###

The word "base" is used to mean "base class, designed for subclassing" except for Google Base-related class files.

### Unit tests ###

Reasonable unit tests should included for all classes of XML elements, services, and queries.

### TODOs ###

To-do items should be rare, and should include the name of the submitter, as in `TODO(grobbins)`

## Deconstructing an Element Object: GDataTextConstruct ##

Most of the classes in the GData framework are extension element objects descended from GDataObject.  This section will examine a typical example, GDataTextConstruct.


GDataTextConstruct holds three strings from the XML: content, lang, and type. The element title varies and has no standard value. The XML for a text construct may look like this:
```
<title type="text">Team Meeting</title>
```

This is easy to represent with a subclass of GDataObject:
```
@interface GDataTextConstruct : GDataObject <NSCopying> {
  NSString *content_;
  NSString *lang_;
  NSString *type_; // text, text/plain, html, text/html, xhtml, or other things
}
```
Note that we use a unique ivar naming convention by appending a trailing underscore.

Each element should have a convenience method allowing an application to easily create an autoreleased instance from scratch.  GDataTextElement provides this:

```
+ (GDataTextConstruct *)textConstructWithString:(NSString *)str {
  GDataTextConstruct *obj = [[[GDataTextConstruct alloc] init] autorelease];
  [obj setStringValue:str];
  return obj;
}
```

Convenience methods for entry and feed classes should also set the namespaces for those elements, as entries and feeds are typically the root elements for XML generation.  For example, the calendar feed object's convenience method performs `[feed setNamespaces:[GDataEntryCalendar calendarNamespaces]]`.

The `init` method for an element may need to set defaults for properties that should never be left uninitialized:
```
- (id)init {
  self = [super init];
  if (self) {
    [self setType:@"text"];
  }
  return self;
}
```

GDataObject subclasses must be able to be initialized from XML as well:
```
- (id)initWithXMLElement:(NSXMLElement *)element
                  parent:(GDataObject *)parent {
  self = [super initWithXMLElement:element
                            parent:parent];
  if (self) {
    
    [self setLang:[self stringForAttributeName:@"xml:lang"
                                   fromElement:element]];
    [self setType:[self stringForAttributeName:@"type"
                                   fromElement:element]];
    [self setStringValue:[self stringValueFromElement:element]];
  }
  return self;
}
```

Note that `initWithXMLElement:parent:` _always_ calls its superclass to parse the XML, such as with `stringForAttributeName:` and `stringValueFromElement:` here. This allows the base class to track which attributes and child elements have been parsed.

The NSObject `dealloc` method will release any objects retained by the class.
```
- (void)dealloc {
  [content_ release];
  [lang_ release];
  [type_ release];
  [super dealloc];
}
```

Elements should also implement the usual NSObject methods `copyWithZone:` and `isEqual:`, in both cases calling through to the base class to permit extensions to be handled properly as well:
```
- (id)copyWithZone:(NSZone *)zone {
  GDataTextConstruct* newText = [super copyWithZone:zone];
  [newText setStringValue:content_];
  [newText setLang:lang_];
  [newText setType:type_];
  return newText;
}

- (BOOL)isEqual:(GDataTextConstruct *)other {
  if (self == other) return YES;
  if (![other isKindOfClass:[GDataTextConstruct class]]) return NO;
  
  return [super isEqual:other]
    && AreEqualOrBothNil([self stringValue], [other stringValue])
    && AreEqualOrBothNil([self lang], [other lang])
    && AreEqualOrBothNil([self type], [other type]);
}
```

Implementing the NSObject method `description` method aids debugging.
```
- (NSString *)description {
  NSMutableArray *items = [NSMutableArray array];
  
  [self addToArray:items objectDescriptionIfNonNil:content_ withName:@""];
  [self addToArray:items objectDescriptionIfNonNil:lang_    withName:@"lang"];
  [self addToArray:items objectDescriptionIfNonNil:type_    withName:@"type"];
  
  return [NSString stringWithFormat:@"%@ 0x%lX: {%@}",
    [self class], self, [items componentsJoinedByString:@" "]];
}
```
XML generation is done by the element's XMLElement method. This always begins with a call to `XMLElementWithExtensionsAndDefaultName:` to create an XML element preloaded with children from extension instances.  Unlike most defined elements, GDataTextConstruct does not have a useful default name; the generated name for the element in the XML will set when the element is created, either from XML or by the application.

Typically, `XMLElement` will add to the NSXMLElement object by calling methods in the GDataObject base class, but this is not mandatory. Unlike in parsing in `initWithXMLElement:parent:`, NSXMLElement methods _may_ be used directly here.

```
- (NSXMLElement *)XMLElement {
  
  NSXMLElement *element = [self XMLElementWithExtensionsAndDefaultName:@"GDataTextConstruct"];

  if ([[self stringValue] length]) {
    [element addStringValue:[self stringValue]];
  }
  [self addToElement:element attributeValueIfNonNil:[self lang] withName:@"xml:lang"];
  [self addToElement:element attributeValueIfNonNil:[self type] withName:@"type"];
  
  return element;
}
```

Finally, elements provide KVC-compatible accessors to their data, using normal Objective-C idioms.

```
- (NSString *)stringValue {
  return content_; 
}

- (void)setStringValue:(NSString *)str {
  [content_ autorelease];
  content_ = [str copy];
}

- (NSString *)lang {
  return lang_; 
}

- (void)setLang:(NSString *)str {
  [lang_ autorelease];
  lang_ = [str copy];
}

- (NSString *)type {
  return type_; 
}

- (void)setType:(NSString *)str {
  [type_ autorelease];
  type_ = [str copy];
}
```