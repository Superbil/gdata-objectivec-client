

# Introduction to Google Data APIs for Cocoa Developers #

[Google Data APIs](http://code.google.com/apis/gdata/index.html) allow client software to access and manipulate data hosted by Google services.

The Google Data APIs Objective-C Client Library is a Cocoa framework that enables developers for Mac OS X and iPhone to easily write native applications using Google Data APIs.  The framework handles

  * XML parsing and generation
  * Networking
  * Sign-in for Google accounts
  * Service-specific protocols and query generation


## Requirements ##

The Google Data APIs Objective-C Client Library requires iOS 3 or Mac OS X 10.5 or later. Applications running on 10.4 may use the "10.4-compatible" branch.

## Example Applications ##

The Examples directory contains example applications showing typical interactions with Google services using the framework.  The applications act as simple browsers for the structure of feed and entry classes for each service.  The WindowController source files of the samples were written with typical Cocoa idioms to serve as quick introductions to use of the APIs.

## Adding Google Data APIs to a project ##

BuildingTheLibrary explains how to add the library to a Mac or iPhone application project.

## Authentication and Authorization ##

**Authentication** is the confirmation of a user's identity via her username, password, and possibly other data, such as captcha answers or 2-step codes provided via mobile phone. Authentication can be done in one of two ways:

  * OAuth 2 is an industry-standard protocol for allowing the server to authenticate the user by providing html login pages. The Google Data API library can use the authorization object provided by the [GTM OAuth 2 controllers](http://code.google.com/p/gtm-oauth2/). See the example applications for Calendar, Docs, or YouTube for a demonstration of OAuth 2 sign-in. OAuth 2 is the recommended method for supporting sign-in.
  * ClientLogin is an older Google-specific protocol for signing in users by asking for their username and password. The application is responsible for secure storage of the password in the system keychain. See "Authentication Errors" below for additional sign-in errors that the application may need to handle. ClientLogin cannot be used for user accounts that have [2-step verification](http://www.google.com/support/accounts/bin/static.py?page=guide.cs&guide=1056283&topic=1056284) enabled unless the user creates an application-specific password. _Note:_ ClientLogin authorization is considered deprecated.

**Authorization** is the use of access tokens for specific requests. Authorization of API requests is handled by the service class.

  * OAuth 2 authorization requires an OAuth2Authentication object be supplied to the service class's `setAuthorizer:` method.
  * ClientLogin authorization requires the username and password be supplied to the service class's `setUserCredentialsWithUsername:password:` method.

## Google Data APIs Basics ##

Servers respond to client GData requests with **feeds** that include lists of **entries**. For example, a request for all of a user's calendars would return a Calendar feed with a list of entries, where each entry represents one calendar.  A request for all events in one calendar would return a Calendar Event feed with a list of entries, with each entry representing one of the user's scheduled events.

Each feed and entry is composed of **elements**.  Elements represent either standard Atom XML elements, or custom-defined GData elements.

Feeds, entries, and elements are derived from **GDataObject**, the base class that implements XML parsing and generation.

Google web application interactions are handled by **service objects**.  A single transaction with a service is tracked with a **service ticket**.

For example, here is how to use the Google Calendar service to retrieve a feed of calendar entries, where each entry describes one of the user's calendars.
```
  service = [[GDataServiceGoogleCalendar alloc] init];

  [service setUserCredentialsWithUsername:username
                                 password:password];
  NSURL *feedURL = [GDataServiceGoogleCalendar calendarFeedURLForUsername:username];

  GDataServiceTicket *ticket;
  ticket = [service fetchFeedWithURL:feedURL
                            delegate:self
                   didFinishSelector:@selector(ticket:finishedWithFeed:error:)];
```
Service objects maintain cookies and track data modification dates to minimize server load, so it's best to reuse a service object for sequences of server requests.

The application may choose to retain the ticket following the fetch call so the user can cancel the service request.  The ticket is valid so long as the application retains it.
To cancel a fetch in progress, call `[ticket cancelTicket]`.  Once the finished or failed callback has been called, the ticket is no longer useful and may be released.

The delegate of the fetch is also retained until the fetch has stopped.

Here is what the callback from a fetch of the calendar list might look like.  This callback example just prints the title of the user's first calendar.
```
- (void)ticket:(GDataServiceTicket *)ticket
finishedWithFeed:(GDataFeedCalendar *)feed
           error:(NSError *)error {

  if (error == nil) {  
    NSArray *entries = [feed entries];
    if ([entries count] > 0) {

      GDataEntryCalendar *firstCalendar = [entries objectAtIndex:0];
      GDataTextConstruct *titleTextConstruct = [firstCalendar title];
      NSString *title = [titleTextConstruct stringValue];
    
      NSLog(@"first calendar's title: %@", title);
    } else {
      NSLog(@"the user has no calendars")
    }
  } else {
    NSLog(@"fetch error: %@", error);
  }
}
```

Service objects include a variety of methods for interacting with the service. Typically, the interactions include some or all of these activities:
  * fetching a feed
  * inserting an entry into a feed
  * updating (replacing) an entry in a feed
  * deleting an entry or a feed
  * performing queries to obtain a subset of the feed's entries
  * performing a [batch operation](http://code.google.com/apis/gdata/batch.html) of inserting, updating, and deleting entries

Feeds and entries usually contain **links** to themselves or to other objects. The library provides convenience methods for retrieving individual links. For example, to retrieve the events for a user's calendar, use a Calendar service object to fetch from the Calendar's "alternate" link:

```
    GDataLink *link = [calendarEntry alternateLink];

    if (link != nil) {
      [service fetchFeedWithURL:[link URL]
                       delegate:self
              didFinishSelector:@selector(eventsTicket:finishedWithFeed:error:)];
    }
```

Typically, the alternate link points to an html or user-friendly representation of the data, though Google Calendar uses it as a link to the event feed for a calendar.

Modifiable feeds may have a **post link**, which contains the URL for inserting new entries into the feed.  Modifiable entries have an **edit link**, which is used to update or delete the entry. Entries that refer to non-XML data, such as photographs, may include an **edit media link** for modifying or deleting the media data.

Both entries and feeds have **self links**, which self-referentially contain the URL of the XML for the entry or feed.  The self link is useful for fetching a complete, current version of an entry prior to updating the entry on the server.

A particularly important link in feeds is the **next link**; it is present when the feed contains only a partial set, or one **page**, of entries from the request.  If the feed's `-nextLink` is non-nil, the client application must perform a new request using the "next" link URL to retrieve the next page of the feed, containing additional entries.

Rather than make a new fetch for each "next" link, the library's service object can follow next links automatically, and return a feed whose entries are the full accumulated set of entries from fetching all pages of the feed (up to 25 pages.)  This can be turned on in the service object with `[service setServiceShouldFollowNextLinks:YES]`. The fetch's ticket then applies to the sequence of http requests needed to obtain the entries from all pages in the feed, so canceling the ticket will cancel the sequence of requests.

Note, however, that feeds spread over many pages may take a long time to be retrieved, as each "next" link will lead to a new http request.  The server can be told to use a larger page size (that is, more entries in each page of the feed) by fetching a query for the feed with a maxResults parameter:

```
      GDataQueryCalendar *query = [GDataQueryCalendar calendarQueryWithFeedURL:feedURL];
      [query setMaxResults:1000];

      GDataServiceGoogleCalendar *service = [self calendarService];
      GDataServiceTicket *ticket;
      ticket = [service fetchFeedWithQuery:query
                                  delegate:self
                         didFinishSelector:@selector(ticket:finishedWithFeed:error:)];
```

Beginning with version 2 of the Google Data API core protocol, each fetched feed and entry has an **ETag** attribute. The ETag is just a server-generated hash string uniquely identifying the version of the entry data. Services may require that the ETag attribute be present when updating or deleting an entry or other file, such as a photo. This prevents the client application from accidentally modifying or deleting the wrong version of the data on the server.

ETag strings are also useful to clients for determining if a feed, entry, or other file on the server has changed.  If the underlying resource data has changed, the ETag is guaranteed to have changed as well.

Service-specific **query** objects can generate URLs with parameters appropriate for restricting a feed's entries. For example, a query could request a feed of Calendar events between specific dates, or of database items of a specified category.  Here is an example of a query to retrieve the first 5 events from a user's calendar:
```
- (void)beginFetchingFiveEventsFromCalendar:(GDataEntryCalendar *)calendar {
    
  NSURL *feedURL = [[calendar alternateLink] URL];
  
  GDataQueryCalendar* query = [GDataQueryCalendar calendarQueryWithFeedURL:feedURL];
  [query setStartIndex:1];
  [query setMaxResults:5];
  
  GDataServiceGoogleCalendar *service = [self calendarService];
  [service fetchFeedWithQuery:query
                     delegate:self
            didFinishSelector:@selector(queryTicket:finishedWithEntries:error:)];
}
```


## Creating GDataObjects from scratch ##

Typically GDataObjects are created by the framework from XML returned from a server, but occasionally it is useful to create one from scratch, such as when uploading a new entry.  This snippet shows how to create a new event to add to a user's calendar:
```
- (void)addAnEventToCalendar:(GDataEntryCalendar *)calendar {
  
  // make a new event
  GDataEntryCalendarEvent *newEvent = [GDataEntryCalendarEvent calendarEvent];
  
  // set a title, description, and author
  [newEvent setTitleWithString:@"Meeting"];
  [newEvent setSummaryWithString:@"Today's discussion"];

  GDataPerson *authorPerson = [GDataPerson personWithName:@"Fred Flintstone"
                                                    email:@"fred.flinstone@example.com"];
  [newEvent addAuthor:authorPerson];
  
  // start time now, end time in an hour
  NSDate *anHourFromNow = [NSDate dateWithTimeIntervalSinceNow:60*60];
  GDataDateTime *startDateTime = [GDataDateTime dateTimeWithDate:[NSDate date]
                                                        timeZone:[NSTimeZone systemTimeZone]];
  GDataDateTime *endDateTime = [GDataDateTime dateTimeWithDate:anHourFromNow
                                                      timeZone:[NSTimeZone systemTimeZone]];

  // reminder 10 minutes before the event
  GDataReminder *reminder = [GDataReminder reminder];
  [reminder setMinutes:@"10"];
  
  GDataWhen *when = [GDataWhen whenWithStartTime:startDateTime
                                         endTime:endDateTime];
  [when addReminders:reminder];

  [newEvent addTime:when];  

  // add it to the user's calendar
  NSURL *feedURL = [[calendar alternateLink] URL];
 
  GDataServiceGoogleCalendar *service = [self calendarService];
  [service fetchEntryByInsertingEntry:newEvent
                           forFeedURL:feedURL
                             delegate:self
                    didFinishSelector:@selector(addTicket:addedEntry:error:)];
}
```

GData services always return to the callback the newest version of an entry that has been inserted or updated, so the methods are called "fetch" even for inserts and updates. When an entry is deleted, however, the callback is passed nil as the object parameter.

## Adding custom data to GDataObject instances ##

Often it is useful to add data locally to a GDataObject. For example, an entry used to represent a photo being uploaded would be more convenient if it also carried a path to the photo's file.

Your application can add data to any instance of a GDataObject (such as entry and feed objects, as well as individual elements) in three ways.

Each GDataObject has methods `setUserData:` and `userData` to set and retrieve a single NSObject. Adding a local path string to a photo entry with `setUserData:` would look like this:

```
GDataEntryPhoto *newPhotoEntry = [GDataEntryPhoto photoEntry];
[newPhotoEntry setUserData:localPathString];
```

An application can set and retrieve multiple objects as named properties of any GDataObject instance with the methods `setProperty:forKey:` and `propertyForKey:`. This is useful when there is more than one bit of data to attach to the object:

```
GDataEntryPhoto *newPhotoEntry = [GDataEntryPhoto photoEntry];
[newPhotoEntry setProperty:localPathString forKey:@"myPath"];
[newPhotoEntry setProperty:thumbnailImage forKey:@"myThumbnail"];
```

Property names beginning with an underscore are reserved by the library and should not be used by applications.



Finally, applications may subclass GDataObjects to add fields and methods.  To have your subclasses be instantiated in place of the standard object class during the parsing of XML following a fetch, call `setServiceSurrogates:`, as demonstrated here:

```
NSDictionary *surrogates = [NSDictionary dictionaryWithObjectsAndKeys:
  [MyEntryPhoto class], [GDataEntryPhoto class],
  [MyEntryAlbum class], [GDataEntryPhotoAlbum class],
  nil];

service = [[GDataServiceGooglePhotos alloc] init];
[service setServiceSurrogates:surrogates];
```

These three techniques only add data to elements _locally_ for the Objective-C code; the data will not be retained on the server. Some services support an [extendedProperty](http://code.google.com/apis/gdata/elements.html#gdExtendedProperty) element which can retain arbitrary data for users.

## Passing objects to fetch callbacks ##

It is also often useful to pass an object to a callback. To retain an object from a fetch call to its callback, use GDataServiceTicket's `setUserData:` or `setProperty:forKey:` methods.  For example:

```
GDataServiceTicket *ticket = [service fetchFeedWithURL:...];
[ticket setProperty:callbackData forKey:@"myCallbackData"];
```

The callback can then access the data:
```
- (void)ticket:(GDataServiceTicket *)ticket finishedWithFeed:(GDataFeedBase *)feed error:(NSError *)error {

  id myCallbackData = [ticket propertyForKey:@"myCallbackData"];
  ...
  }
```


## Batch requests ##

Some services support batch requests, allowing many insert, update, or delete operations to be performed with a single http request.

Services which allow batch operations on a feed's entries will provide a batch link in the feed:

```
NSURL *batchURL = [[feed batchLink] URL];
```

To execute a batch request, create an empty feed for the appropriate service, such as:

```
GDataFeedContact *batchFeed = [GDataFeedContact contactFeed];
```

Then add entries to insert, update, and delete to the batch feed. For updates and deletes, add entries that were previously fetched rather than ones created from scratch, as the fetched entries will have additional elements (edit links or ETags) that the server may need to check that modifications are occurring on the expected entry versions.

Using the first five entries from a fetched contacts feed in a batch feed might look like:

```
NSRange entryRange = NSMakeRange(0, 5);
NSArray *entries = [[contactFeed entries] subarrayWithRange:entryRange];
[batchFeed setEntriesWithEntries:entries];
```

If the same operation applies to all entries, add that operation to the batch feed instance:

```
GDataBatchOperation *op;
op = [GDataBatchOperation batchOperationWithType:kGDataBatchOperationDelete];
[batchFeed setBatchOperation:op];    
```

If different entries require different operations, add operation elements to the individual entries of the batch feed.

Optionally, each entry may be assigned a batch ID.  A batch ID is an arbitrary string, created by your application, that is copied by the server into the entries of the batch results feed.  This lets your code match the results to the entries of the original batch feed.  The server merely copies the ID string; the string's contents are up to the application.

```
static unsigned int staticID = 0;
NSString *batchID = [NSString stringWithFormat:@"batchID_%u", ++staticID];
[entry setBatchIDWithString:batchID];
```


As with other kinds of fetches in the library, objects may be passed to the batch callbacks as properties or userData on the ticket:

```
GDataServiceGoogleContact *service = [self contactService];
GDataServiceTicket *ticket;

ticket = [service fetchFeedWithBatchFeed:batchFeed
                         forBatchFeedURL:batchURL
                                delegate:self
                       didFinishSelector:@selector(batchDeleteTicket:finishedWithFeed:error:)];

[ticket setProperty:@"my data"
             forKey:kMyDataKey];
```

The callback for a batch operation receives a feed that includes entries as the response for each operation.  Each result entry has a status code element, and may have the optional ID string.  If batch processing was halted at an entry, the corresponding response entry will have a GDataBatchInterrupted element.

```
- (void)batchTicket:(GDataServiceTicket *)ticket
   finishedWithFeed:(GDataFeedBase *)resultsFeed
              error:(NSError *)error {
  if (error == nil) {    
    for (int idx = 0; idx < [resultsFeed count]; idx++) {
    
      GDataEntryContact *entry = [resultsFeed objectAtIndex:idx];
    
      NSString *batchIDString = [[entry batchID] stringValue];
    
      GDataBatchStatus *status = [entry batchStatus];
      int statusCode = [[status code] intValue];
      NSString *statusReason = [status reason];
    
      GDataBatchInterrupted *interrupted = [entry batchInterrupted];
      if (interrupted != nil) {
        // batch processing was interrupted, probably by invalid XML;
        // no further entries were processed, and no more result entries
        // are provided
      }
    }
  } 
}
```

To ensure reasonable response times, a service may impose a limit on the number of entries in a single batch. See the service's documentation for specific limits on the size of a batch.

The [batch processing documentation](http://code.google.com/apis/gdata/batch.html) has additional information on batch requests.

## Uploading files ##

Some services allow uploading of a file when inserting an entry into a feed.  Uploading requires setting the upload data and MIME type in the entry. Some services also require a **slug** be provided as the file name used to store the data on the server.

Some services just allow uploads to be done in a single fetch. Those use the URL of the feed's postLink for inserting a new entry and its data, and the URL of an entry's editLink or editMediaLink for replacing or deleting the data.

Services that support uploads of very large files (Google Docs, Google Photos, and YouTube) use a **chunked upload** protocol. Those services provide a separate uploadLink in feeds, and an uploadEditLink in entries, but otherwise use the same fetch calls for uploading files. In addition, chunked uploads may be paused and resumed by invoking the ticket's `pauseUpload` and `resumeUpload` methods.

This snippet shows the basic steps for uploading a spreadsheet document using the Google Docs API.

```
GDataEntrySpreadsheetDoc *newEntry = [GDataEntrySpreadsheetDoc documentEntry];
  
NSString *path = @"/mySpreadsheet.xls";
NSData *data = [NSData dataWithContentsOfFile:path];
if (data) {
  NSString *fileName = [path lastPathComponent];
  
  [newEntry setUploadSlug:filename];
  [newEntry setUploadData:data];
  [newEntry setUploadMIMEType:@"application/vnd.ms-excel"];

  NSString *title = [[NSFileManager defaultManager] displayNameAtPath:path];
  [newEntry setTitleWithString:title];
  
  NSURL *uploadURL = [GDataServiceGoogleDocs docsUploadURL];
  // services without chunked upload support typically
  // allow uploading to the feed's postLink, like
  // NSURL *uploadURL = [[feed postLink] URL];

  ticket = [service fetchEntryByInsertingEntry:newEntry
                                    forFeedURL:postURL
                                      delegate:self
                             didFinishSelector:@selector(uploadTicket:finishedWithEntry:error:)];
}
```

To avoid reading large files into memory during chunked uploads, uploading can also be done using NSFileHandle rather than NSData:

```
  NSFileHandle *bigFH = [NSFileHandle fileHandleForReadingAtPath:bigFilePath];
  [newEntry setUploadFileHandle:bigFH];
```

### Upload progress monitoring ###

When uploading large blocks of data, such as photos or videos, your application can request periodic callbacks to update a progress indicator.  To receive the periodic callback, set an upload progress selector in the service, such as:

```
SEL progressSel = @selector(ticket:hasDeliveredByteCount:ofTotalByteCount:);
[service setServiceUploadProgressSelector:progressSel];

// then do the fetch
//   GDataServiceTicket *ticket = [service fetch...];

// If future tickets should not use the progress callback, 
// set the selector in the service back to nil

[service setServiceUploadProgressSelector:nil];
```

The callback is a method with a signature matching this:
```
- (void)ticket:(GDataServiceTicket *)ticket 
   hasDeliveredByteCount:(unsigned long long)numberOfBytesRead 
        ofTotalByteCount:(unsigned long long)dataLength {

}
```

**_Note:_** In library versions 1.8 and earlier, the progress callback's first argument is a pointer to a GDataProgressMonitorInputStream, and the ticket is available from the stream's `-monitorSource` method.

## Status 304 and service data caching ##

GData servers provide a "Last-Modified" header with their responses.  The service object remembers the header, and provides it as an "If-Modified-Since" request the next time the application makes a request to the same URL.  If the request would have the same response as it did previously, the server returns no data to the second request, just status 304, "Not modified".

Your service delegate will see the "Not modified" response in its callback method.  The application handles it like this:

```
- (void)ticket:(GDataServiceTicket *)ticket finishedWithFeed:(GDataFeedBase *)feed error:(NSError *)error {
  if (error == nil) {
    // fetch succeeded
  } else {
    // fetch failed
    if ([error code] == kGTMHTTPFetcherStatusNotModified) { // status 304
      // no change since previous request
    } else {
      // some unexpected error occurred
    }
  }
}
```

The service object can optionally remember the dated responses in a cache and provide them to the application instead of calling the failure method.  To enable the caching, the application should call
```
[service setShouldCacheDatedData:YES];
```

The service will thereafter call the fetch callback methods with duplicates of the original response rather than with a status 304 error.  You can call `[service clearLastModifiedDates]` to purge the cache, or `[service setShouldCacheDatedData:NO]` to purge and disable the future caching.

## Fetching during modal dialogs ##

The networking code in GDataService classes is based on NSURLConnection, and as in NSURLConnection, callbacks are deferred while a modal dialog is displayed.  Under Mac OS X 10.5 or later, you can specify run loop modes to allow networking callbacks during modal dialogs:

```
NSArray *modes = [NSArray arrayWithObjects:
  NSDefaultRunLoopMode, NSModalPanelRunLoopMode, nil];
[service setRunLoopModes:modes];
```

## Authentication errors ##

If your application authenticates the user via [OAuth 2 controller](http://code.google.com/p/gtm-oauth2/), then no additional handling of authentication errors is needed: invalid username and password, captchas, and 2-step verification are handled automatically.

[Authentication](http://code.google.com/apis/gdata/auth.html) by asking the user for Google account username and password is called ClientLogin. Authenticating the user via ClientLogin with Google's services is handled mostly transparently by the framework.  If your application sends the  `setUserCredentialsWithUsername:password:` message to the service object, the service object will sign in prior to fetching the requested object.

The most common error during authentication is an invalid username or password.

### 2-step verification ###

If a user's account requires [2-step verification](http://goo.gl/8bHWV), the user must supply an [application-specific password](http://goo.gl/bogZI) rather than the account password. That case can be detected by inspecting the callback error:
```
if ([[[error userInfo] authenticationInfo] isEqual:kGDataServerInfoInvalidSecondFactor]) {
  // user provided account password but
  // should provide application-specific password 
}
```

With version 1.11 of the library and earlier, use this criteria instead:
```
if ([[[error userInfo] objectForKey:@"Info"] isEqual:@"InvalidSecondFactor"]) {
  // user provided account password but
  // should provide application-specific password 
}
```

### Captchas ###

Occasionally, the servers may also request that the user solve a [captcha](http://www.captcha.net/), a visual puzzle. It is **optional** for your application to handle captcha requests. One easy way to handle them is to just open Google's unlock captcha web page in the user's web browser.

```
NSURL *captchaUnlockURL = [NSURL URLWithString:@"https://www.google.com/accounts/DisplayUnlockCaptcha"];
[[NSWorkspace sharedWorkspace] openURL:captchaUnlockURL];
```

For the best user experience in handling captcha requests, your application may download and display the captcha image to the user, and wait for the user to provide the answer. If the user offers an answer, put the captcha token and the user's answer into the service object with `setCaptchaToken:captchaAnswer:`, and retry the fetch.

To handle captchas, the fetch callback method might look something like this:

```
- (void)ticket:(GDataServiceTicket *)ticket finishedWithFeed:(GDataFeedBase *)feed error:(NSError *)error {
  if (error == nil) {
    // fetch succeeded
  } else {
    // fetch failed
    if ([error code] == kGDataBadAuthentication) {
    
      NSDictionary *userInfo = [error userInfo];
      NSString *authError = [userInfo authenticationError];
    
      if ([authError isEqual:kGDataServiceErrorCaptchaRequired]) {
        // URL for a captcha image (200x70 pixels)
        NSURL *captchaURL = [userInfo captchaURL];
        NSString *captchaToken = [userInfo captchaToken];
      
        // a synchronous read of the image is simple, as shown here,
        // but to be nice to users, you can use GTMHTTPFetcher for
        // an easy asynchronous fetch of the data instead
        NSData *imageData = [NSData dataWithContentsOfURL:captchaURL];
        if (imageData) {
          NSImage *image = [[[NSImage alloc] initWithData:imageData] autorelease];
 
          [self askUserToSolveCaptchaImage:image 
                                 withToken:captchaToken];

          // pass the token and user's captcha answer later to the service, like
          //   [service setCaptchaToken:captchaToken captchaAnswer:theUserAnswer]
          // prior to retrying the fetch
        }
      } else {
        // invalid username or password
      }
    } else {
      // some other error authenticating or retrieving the GData object
      // or a 304 status indicating the data has not been modified since it
      // was previously fetched
    }
  }
}
```

**_Tip_**: to force a captcha request from the server for testing, provide an invalid e-mail address as the username several times in a row.

Documentation on authentication for Google data APIs is available [here](http://code.google.com/apis/gdata/auth.html).

## Automatic retry of failed fetches ##

GData service classes and the GTMHTTPFetcher class provide a mechanism for automatic retry of a few common network and server errors, with appropriate increasing delays between each attempt.  You can turn on the automatic retry support for a GData service by calling `[service setIsServiceRetryEnabled:YES]`.

The default errors retried are http status 408 (request timeout), 503 (service unavailable), and 504 (gateway timeout), and NSURLErrorTimedOut.  You may specify a maximum retry interval other than the default of 10 minutes, and can provide an optional retry selector to customize the criteria for each retry attempt.

## Proxy Authentication ##

In corporate or institutional settings where a password-protected proxy is in use, a proxy error may show up in the failure callback.  It would have constant error domain and code values, as shown here:
```
- (void)fetchTicket:(GDataServiceTicket *)ticket
   finishedWithFeed:(GDataFeedBase *)feed
              error:(NSError *)error {
  if (error == nil) {
    // fetch succeeded
  } else {
    // fetch failed
    if ([error code] == kGTMHTTPFetcherErrorAuthenticationChallengeFailed
        && [[error domain] isEqual:kGTMHTTPFetcherErrorDomain]) {

      NSURLAuthenticationChallenge *challenge;
      challenge = [[error userInfo] objectForKey:kGTMHTTPFetcherErrorChallengeKey];
    }
  }
  ...
}
```
If you want to handle such errors, use the
challenge object to display a dialog showing information about the host and requesting
account and password credentials for the proxy.  The proxy credentials can then
be passed to the ticket's authentication fetcher:
```
ticket = [service fetchFeed...];
if (proxyAccountName && proxyPassword) {
  NSURLCredential *cred;
  cred = [NSURLCredential credentialWithUser:proxyAccountName
                                    password:proxyPassword
                                 persistence:NSURLCredentialPersistencePermanent];
  [[ticket authFetcher] setProxyCredential:cred];
}
```

## Logging http server traffic ##

Debugging GData transactions is often easier when you can browse the XML being sent back and forth over the network.  To make this convenient, the framework can save copies of the server traffic, including http headers, to files in a local directory.  Your application should call

```
[GTMHTTPFetcher setLoggingEnabled:YES]
```

to turn on logging.  Normally, logs are written to the directory GTMHTTPDebugLogs in the current user's Desktop folder, though the path to another folder can be specified with the `+setLoggingDirectory:` method.

In the iPhone simulator, the default logs location is the user's home directory. On the iPhone device, the default location is the application's documents folder on the device.

To view the most recently saved logs, use a web browser to open the symlink named My\_App\_Name\_http\_log\_newest.html (for whatever your application's name is) in the logging directory.  Note that Camino and Firefox display XML in a more useful fashion than does Safari.

**_Tip:_** providing a convenient way for your users to enable logging is often helpful in diagnosing problems when using the API.

## Service Introspection ##

For each feed, services provide an XML document that describes capabilities of the feed, typically including the types of media data that is accepted for upload.  Applications can dynamically adjust how they use a feed by retrieving the feed's **service document**.

For instance, given the URL to a feed for a user's photo album, the service document for the feed can be fetched like this:

```
GDataQueryGooglePhotos *introspectQuery;
introspectQuery = [GDataQueryGooglePhotos photoQueryWithFeedURL:albumFeedURL];
[introspectQuery setResultFormat:kGDataQueryResultServiceDocument];

GDataServiceTicket *ticket;
ticket = [photosService fetchFeedWithQuery:introspectQuery
                                  delegate:self
                         didFinishSelector:@selector(introspectTicket:finishedWithServiceDocument:error)];
```

The service document includes a **workspace** that has a **collection** describing the feed. Feeds whose entries refer to other feeds will have one collection for each entry's feed.

For an album feed, the MIME types that can be uploaded would be obtained from the fetched service document this way:

```
- (void)introspectTicket:(GDataServiceTicket *)ticket
finishedWithServiceDocument:(GDataAtomServiceDocument *)serviceDoc
                      error:(NSError *)error {
  if (error == nil) {
    GDataAtomCollection *collection = [[serviceDoc primaryWorkspace] primaryCollection];
    NSArray *theMIMETypes = [collection serviceAcceptStrings];
    ...
  }
}
```

## Performance and Memory Improvements ##

Once your application successfully works with Google Data APIs, review the tips on the [Performance Tuning](http://code.google.com/p/gdata-objectivec-client/wiki/PerformanceTuning) page.

## Questions and Comments ##

**If you have any questions or comments** about the library or this documentation, please join the [discussion group](http://groups.google.com/group/gdata-objectivec-client).