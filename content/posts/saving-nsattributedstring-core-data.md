---
title: "An unfortunate surprise when saving NSAttributedString in Core Data"
date: 2020-11-29T02:06:36+02:00
draft: false
---
### Introduction
Saving an attributed string in `Core Data` isn't a difficult job. While the `NSAttributedString` type it's not listed in the entity attribute's type list, you can: 
1. Set attribute's type to `Transformable`
2. Create a subclass of `NSSecureUnarchiveFromDataTransformer`
3. Go back to `Core Data Model` editor, and update `Transformer` field of the attribute, with the class name from the previous step.

That's everything you should usually do to store an attributed string in `Core Data`. However, I've encountered an interesting crash, after implementing this. 

### Backstory
One of my app's features allows users to fetch text from a PDF file via `PDFSelection`. The selection returns selected text in plain form or attributed form. I'm using the attributed version of the text because I need some of its styling. For testing purposes, I have 2 PDFs, and both proved the feature works correctly. 

However, a few days ago I added a new PDF, which started to crash my app. Looking at the crash logs I saw the following: (a fragment)

` *** Assertion failure in -[UICGColor encodeWithCoder:], UIColor.m:2252
2020-11-29 17:27:58.693467+0200 EpicNote[40455:2797002] [error] error: SQLCore dispatchRequest: exception handling request: <NSSQLSaveChangesRequestContext: 0x280d4db00> , <shared NSSecureUnarchiveFromData transformer> threw while encoding a value. with userInfo of {
  NSUnderlyingError = "Error Domain=NSCocoaErrorDomain Code=4866 \"Caught exception during archival: Only RGBA or White color spaces are supported in this situation.\n(\n\t0  CoreFoundation           0x00000001841639e8 96F8386D-D88A-3C89-A323-A17975C3317F + 1157608\n\t1  libobjc.A.dylib           0x0000000197b14b54 objc_exception_throw + 56\n\t2  CoreFoundation
  `

First thing to notice is the crash reason:
> Assertion failure in -[UICGColor encodeWithCoder:], UIColor.m:2252; 
shared NSSecureUnarchiveFromData transformer threw while encoding a value.

The error message was:
> Caught exception during archival: Only RGBA or White color spaces are supported in this situation.

### Troubleshooting
In hindsight I should better analyze the crash log details, it contained a crucial hint that I initially missed. About that a bit later.

I made a shortlist of the possible sources of the crash:

- `UIColor` created using `init(patternImage image: UIImage)`. Maybe color build based on an image was corrupted.
- `NSSecureUnarchiveFromDataTransformer`. Maybe I've misimplemented the subclass, although based on the error message there were few chances.
- `NSSecureCoding`. Check classes that are storing the color and know how to serialize/deserialize a color object, again maybe something was misimplemented.

To check the aforementioned list, I needed a few good hours, mixed with a few breaks. I've also started to question my life choices, whether programming is still the job I want to do ("this damn thing should work, it was working earlier!"). In the end, everything was set up correctly, nothing was misplaced.

I decided to re-read the crash log line by line, to see if I didn't miss anything. Well, of course, I did!
The crash log length was pretty impressive and after the first 5 lines, it looked homogenous. But not long after I started to analyze again the error message I saw the life-saving `DataModelKit` which is one of my frameworks. On the same line was living this scribble `s12DataModelKit35DMKAttributedStringValueTransformerC018reverseTransformedF0yypSgAEF`. Once I isolated `DMKAttributedStringValueTransformer` from that mess, I instantly realized it was a color value from the attributed string. To prove my assumption right, I checked the attributed string received from the problematic PDF, for the color attributes: 

> NSColor = "kCGColorSpaceModelCMYK 0 0 0 1 1 ";

Oh, so it's a CMYK color space. This is why Core Data complained. It wants only colors with RGB or White color space.

I've also checked another PDF for this and the result was:

> NSColor = "kCGColorSpaceModelMonochrome 0 1 ";

This one has a white color space. This works fine.

### Solution

No wonder this issue came as a huge surprise. I expected that a class as `NSAttributedString` that conforms to `NSSecureCoding` have no problems encoding/decoding it's values. Otherwise, it's a dangerous behavior. In my case attributed strings are provided by the `PDFKit` APIs, and because attributed strings can have lots of attributes, Core Data may complain about some of them having the "wrong" value. Which one? I don't know. The future seems to be no bright.

The solution to this would be to sanitize attributed strings. Remove unnecessary attributes aka keep only the attributes you need. Create a new attributed string with the string from the source attributed string. Read attributes' values you're interested in (from source attributed string) and add them in the new attributed string.

In my case, I've removed redundant attributes and set the value of the "star of the show" - `.foregroundColor` attribute to `UIColor.black` to avoid colors with non-supported color space by the `NSCoder`.
