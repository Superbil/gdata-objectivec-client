

# Performance Tuning in the Google Data APIs Objective-C Library #

## Setting the feed size ##

Feeds may contain zero, one, or many entries; the default "page size" for a feed is up to the server. When a feed does not contain all entries, it has a link to the next page of the feed.

The service class provides a method to hide the paging of feeds. Calling `[service setServiceShouldFollowNextLinks:YES]` makes the library accumulate the entries from successive pages of a feed.  Though this hides the work of following "next" links, it also hides the cost, which may be several seconds to fetch each page of the feed.

Sometimes, the default page size is far different from the typical number of entries in a feed. For example, while a default page size of 20 entries may be enough for a feed of a user's calendars, it's typically far smaller than the number of events on a user's calendar, which may in the hundreds or thousands.

When fetching feeds, you should have some idea of the total number of entries typically expected for that feed. For example, it is reasonable to anticipate that a user has fewer than 20 calendars, or more than 1000 events in a calendar. By fetching a feed with a query specifying a reasonably large number of entries to be returned at once, such as `[query setMaxResults:1000]`, you can reduce the number of network requests, speeding up the application.

iPhone applications fetching feeds may also be concerned about retrieving too many entries for a feed, since entries can each take up quite a bit of memory, and parsing feeds with many entries can be slow.  When a feed is likely to return more entries than the application wants at once, you can turn off the service's `setServiceShouldFollowNextLinks:` setting and specify an appropriate maximum number of entries desired in the feed with a query's `setMaxResults:` method.

Remember that turning off automatic following of "next" links will leave it up to the application to check for and follow a feed's `-nextLink` to get all the entries.

## Clearing the Cache ##

Google Data API servers look for an "If-modified-since" header on requests. That header can enable the server to avoid sending the same data repeatedly to an application. When a server sees that the header matches the "Last-modified" date on the previous response to the same request, the server returns a status 304, "Not Modified" error rather than returning the data.

The Objective-C library automatically tracks the "Last-modified" dates for all fetched entries and feeds, and provides those dates to the server in future request fetches. So every application must either expect and handle 304 status responses on fetches, or else should turn on caching by calling `[service setShouldCacheDatedData:YES]` on the service instance.  With caching of dated data enabled, the library will hide the 304 responses and return to the application cached copies of the previous server response data with a 200, "Success" response status.

_New since release 1.8_: The accumulated cache data is limited in size (15 MB on Mac OS X, 1 MB on iPhone).  Once a cached response is removed from the cache, a repeated request will not have the "Last-modified" header when sent to the server, so no 304 status response will occur.

Applications that do not expect to fetch again soon may explicitly clear the cache to free up memory by calling `[service clearLastModifiedDates]`.

## Using Partial Responses and Updates ##

Because the Atom protocol underlying Google Data APIs is verbose, entries often include far more elements than an application cares to use. This wastes time in transmission and parsing, and memory used by the GDataObjects.

Some services (Calendar, Picasa Web Albums, and YouTube) support **partial response** and **partial update**. To get a partial response, the application specifies just what elements it wants included in the feed or its entries. A subset of [XPath syntax](http://www.zvon.org/xxl/XPathTutorial/General/examples.html) is used to select the fields of the feed and entries.

For example, this code snippet retrieves calendar entries with only the `atom:title` and `gd:when` elements, and the standard `gd` attributes (`gd:etag`, `gd:kind`, `gd:fields`):
```
GDataQueryCalendar *query = [GDataQueryCalendar calendarQueryWithFeedURL:feedURL]; 
[query setFieldSelection:@"entry(@gd:*,title,gd:when)"];
[query fetchFeedWithQuery:query ...
```
Partial responses can even specify which of many similar elements are desired. This query selection would retrieve a feed's entries with just the title and the edit link of each entry, along with the standard `gd` attributes:
```
GDataQueryCalendar *query = [GDataQueryCalendar calendarQueryWithFeedURL:feedURL]; 
[query setFieldSelection:@"entry(@gd:*,title,link[@rel=\"edit\"])"]; 
[query fetchFeedWithQuery:query ...
```

Do always include the GData entry attributes ("`@gd:*`") in partial queries since the parser may need the `gd:kind` attributes to determine what class of entry to instantiate.

To update just specific portions of an entry, add a field selection to the entry object. This example shows how to change only the title of a calendar entry, ignoring any other changes in the entry:
```
[calendarEntry setTitleWithString:newName]; 
[calendarEntry setFieldSelection:@"title"]; 
[service fetchEntryByUpdatingEntry:calendarEntry ... 
```
Additional information about partial response and update support is available in [the protocol documentation](http://code.google.com/apis/gdata/docs/2.0/reference.html#PartialResponse).

**_Note:_** The Picasa Web Albums API and YouTube API do not yet support the `gd:kind` attribute, so partial queries should include the category element to enable the library to identify the class of feed or entry to instantiate, like `[query setFieldSelection:@"category,entry(@gd:*,category, ...`

## Optimizations for iPhone Applications ##

[Development with the iPhone SDK](GDataObjCIntroduction#Development_with_the_iPhone_SDK.md) explains how to add the library to your iPhone application project.

### Ignoring Unknown XML ###

iPhone applications can speed up parsing of feeds and reduce memory usage during parsing of feeds by calling

```
  [service setShouldServiceFeedsIgnoreUnknowns:YES];
```

After this call, the library will not attempt to retain the unexpected child elements and attributes for any parsed objects in a feed.  This improves parsing performance and reduces memory usage when parsing large feeds (such as feeds having over 100 entries.)  The benefit is not noticeable in desktop applications.

The downside is that the full entries in the feed fetched this way cannot be directly used for updates (but may be used for partial updates, as described above.) Attempting to update a full entry retrieves when unknown elements were ignored will make the library throw an assert.

To update a full entry from a feed that ignores the unknown XML, fetch a complete copy of the entry using the self link before doing the update fetch. Here's how it might look to fetch a complete contact entry, given the entry from the feed.

```
- (void)retrieveCompleteEntryForEntry:(GDataEntryContact *)entry
                             userData:(id)userData {
 
  GDataServiceGoogleContact *service = [self contactService];

  // fetch a complete copy of the entry, including unknown XML, by using the self link
  NSURL *entryURL = [[entry selfLink] URL];
  GDataServiceTicket *ticket;
  ticket = [service fetchEntryWithURL:entryURL
                             delegate:self
                    didFinishSelector:@selector(fetchEntryTicket:finishedWithEntry:error:)];
   
  [ticket setUserData:userData];
}

- (void)fetchEntryTicket:(GDataServiceTicket *)ticket
       finishedWithEntry:(GDataEntryContact *)completeEntry
                   error:(NSError *)error {
  if (error == nil) {
    // now we can modify the complete entry and update it on the
    // server by calling -fetchContactEntryByUpdatingEntry:
  }
}
```

To fetch several complete entries at once, use a batch fetch with a query batch operation.  Each entry in the batch feed should contain the identifier of the entry to be updated.