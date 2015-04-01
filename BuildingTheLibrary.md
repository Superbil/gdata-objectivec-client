

# Adding the Google Data APIs to a Project #

The Google Data APIs Objective-C Client Library is provided as a built framework, suitable for inclusion in a Mac application bundle's Frameworks folder, and as a static library for iPhone applications.  Alternatively, the sources may also be compiled directly into a Mac or an iPhone application.

## Checking Out Sources ##

Check out the top-of-trunk sources using the command line sources shown [here](http://code.google.com/p/gdata-objectivec-client/source/checkout).

## Linking to the Mac OS X Framework ##

To add the framework to an Xcode project, drag GData.framework to the project's Linked Frameworks source group, then drag the GData framework from the  Linked Frameworks group folder to the Link Binary With Library phase inside of the application target.

Source files referring to GData objects should include either the full GData headers as

`#import "GData/GData.h"`

or the header for a specific service, such as

`#import "GData/GDataCalendar.h"`

To facilitate debugging, you may choose to include the GData.xcodeproj project file directly in your application project as a cross-project reference.  The example applications show how to include a reference to the GData framework project file in an Xcode project.

**_Tip_**: if Xcode's debugger is ignoring breakpoints set in the framework, turn off the "Load symbols lazily" option in Xcode's Debugging preferences.

## Linking to the iPhone Static Library ##

_**Tip**: Developer hoishing has published helpful tips, including photographs, for building the static library in Xcode 4 in [his blog](http://hoishing.wordpress.com/2011/08/23/gdata-objective-c-client-setup-in-xcode-4/)._

The library project includes a target for building a static library for iPhone and iPod Touch apps; this avoids dragging individual library source files into the iPhone application's project.

To build with the static library, drag the GData project file itself into an iPhone project to make a cross-project reference, and add the GDataTouchStaticLib target as a dependency for building the app.

Drag the static library target from under the GData.xcodeproj cross-project reference in the application project to the application target's "Link Binary With Libraries" build phase.

Next, add the [ObjC link option](http://developer.apple.com/mac/library/qa/qa2006/qa1490.html) to the application target's build settings, along with the libxml2 and [all\_load](http://developer.apple.com/library/mac/#qa/qa2006/qa1490.html) flags:

> Other Linker Flags: `-ObjC -lxml2 -all_load`

Also add the compile flags described below in "Removing Unneeded Code".

The static library build creates a directory of header files (in its build products directory, `~/Library/Developer/Xcode/DerivedData`) that should be dragged into your application project. With the static library linked into your project, refer directly to the headers by omitting the framework name, like

`#import "GDataCalendar.h"`

**_Note:_** If you include the OAuth 2 sign-in classes in the library, your application will need to link to Security.framework and SystemConfiguration.framework. To avoid including unneeded library code, see _Removing Unneeded Code_ below.

## Compiling the Source Files Directly into a Mac or iPhone Application ##

Rather than link to the GData framework, you can compile the GData library sources directly into your own project. To do this, drag the GData Sources source group from the GData Xcode project into your project's window (add by reference, not by copying the files.)

You can delete the references to the client services (Calendar, Contacts, Spreadsheet, and so on) that are not needed by your application, though they will not be compiled if you set the compiler flags described below in "Removing Unneeded Code."


If you compile the project's source files directly into your own project file, set this build setting:

> C Language Dialect: `C99 [-std=c99]`

Search the build settings for "c99" to find the setting. If it's not present as a build  option, and if a compile error requires c99, then set the equivalent user-defined setting:

> `GCC_C_LANGUAGE_STANDARD=c99`

For just the Debug configuration of your target, add this compiler definition to ensure that the library's debug-only code is included:

> Other C Flags: `-DDEBUG=1`

Or, if the Other C Flags setting is not available in your target's build options, set the equivalent user-defined setting:

> `OTHER_CFLAGS=-DDEBUG=1`


With the source files compiled directly in your project, refer directly to the headers by omitting the framework name, like

`#import "GDataCalendar.h"`

**_ARC Compatibility_**

When the library source files are compiled directly into a project that uses ARC, then ARC must be disabled specifically for the library sources.

To disable ARC for source files in Xcode 4, select the project and the target in Xcode. Under the target "Build Phases" tab, expand the Compile Sources build phase, select the library source files, then press Enter to open an edit field, and type  `-fno-objc-arc`  as the compiler flag for those files.

**_Notes for iPhone apps compiling the source files directly_**

Be sure that the files GDataXMLNode.m and GDataXMLNode.h in the Common/Optional/XMLSupport group are included in your project.  They are required for iPhone builds.

iPhone applications also need these build settings in the project or target:

> Header Search Paths: `/usr/include/libxml2`

> Other Linker Flags:  `-lxml2`

## Removing Unneeded Code ##

However you build the library, it is worthwhile to remove code for the services that your application will not be using. To remove unneeded code, set these compile flags in the  project or target build settings of the GData.xcodeproj project file. When compiling the sources directly into your application, set these flags in your project's settings.

The conditional flags in most source files of the library look something like this:
```
#if !GDATA_REQUIRE_SERVICE_INCLUDES || GDATA_INCLUDE_CONTACTS_SERVICE
```
So in either the project's settings for compile flags or in your project's config file, set
`GDATA_REQUIRE_SERVICE_INCLUDES` to 1, along with the services you are using.
For example,
> Other C Flags: `-DGDATA_REQUIRE_SERVICE_INCLUDES=1 -DGDATA_INCLUDE_CONTACTS_SERVICE=1`

If your project compiles the OAuth 2 classes for authentication, also define the conditional `-DGTM_INCLUDE_OAUTH2=1` in Other C Flags.

Do remember to set build settings so they apply to both Debug _and_ Release configurations.

There is a conditional set in the static library target as a reminder to developers to define the needed services.  For your project, replace or delete the definition
```
-DGDATA_INCLUDE_nameServiceHere_SERVICE=1
```
in the Other C Flags section of the static library target's Release configuration.