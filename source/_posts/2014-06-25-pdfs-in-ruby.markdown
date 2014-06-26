---
layout: post
author: Michael B
title: "PDF's in Ruby"
date: 2014-06-25 21:18:36 +1000
comments: true
categories: 
---

Avoid using PDF's in your application. There are no great solutions to PDF generation in general, and Ruby does not have any perfect options. If you really need PDF's, this is the landscape of options, and my suggestion.

<!-- more -->

##HTML to PDF

PDF generation in Ruby, and in general, comes down to libraries that turn HTML + CSS into PDF's, and libraries that abstract the PDF standard into programmatic creation of documents.

The gold standard of HTML to PDF is [PrinceXML](http://www.princexml.com/). By all accounts this is a great product, used by big corporates and universities. However if the XML in the title didn't give it away, this is very much an 'enterprise' product, starting at $3800 for a single license!

The open source alternative is [wkhtmltopdf](http://wkhtmltopdf.org/). Two gems that leverage this library are [PDFKit](https://github.com/pdfkit/pdfkit) and [Wicked PDF](https://github.com/mileszs/wicked_pdf).

At first these libraries seem great, code in what you know, leverage existing controllers and even views. Ultimately though its a fairly bad abstraction, and PDF rendering can be slow and unreliable. This is not a problem unique to Ruby and I suspect it's why PrinceXML can still get away with their pricing.

The Ruby alternative is [Prawn](http://prawnpdf.org/api-docs/), a gem with a DSL for building documents.

##Go the raw Prawn

Prawn does have a learning curve, but it performs well, doesn't rely on external binaries and gives full access to layout and paging. The prawn DSL is described fairly well in the [manual](http://prawnpdf.org/manual.pdf) (generated of course in Prawn). Be careful using code examples on Github and older blog posts, Prawn has had some major API changes over time and a lot of old code examples floating around wont work. 

For example Prawn has template functionality, that lets you use an existing PDF as a template. However when template PDF's are too large this feature crashes silently rendering a blank PDF.

If you stick to the methods used in the manual and learn the DSL Prawn is a flexible and reliable solution. The Prawn way of building documents makes a lot of sense and the simplicity of having no external requirements makes integrating into all kinds of Ruby projects easy.


##General PDF Weirdness

Some things to watch out for

* Merging PDF's - there are absolutely no Ruby gems capable of doing this reliably, shelling out to something like [PDFtk](http://www.pdflabs.com/tools/pdftk-the-pdf-toolkit/) does work but is far from ideal.
* Google Chrome PDF Viewer Issue - This is a really odd issue that I eventually found in a google support ticket. The PDF viewer in chrome cant copy any text containing a line ending in '-' (and possibly some other characters). This issue doesn't exist in any other PDF viewer and is unlikely to effect most applications, but its a good example of the fun world of PDF's!
* Editing existing PDF's - don't even try, as mentioned previously Prawn claims to do this but it doesn't work very well

















