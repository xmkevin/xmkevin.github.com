---
layout: post
title: "Updating storyboard strings"
description: ""
category: 
tags: [storyboard,strings]
---
{% include JB/setup %}

I use Storyboard to present UI in iOS applications, all work fine. However, when I update the UI, it's hard to update the releated localization strings, I usually don't know what strings should be added and translated. It must be a easier way to do this.

##Base internationalization

Let's take a look at the Apple definition: 
	"Base internationalization separates user-facing strings from .storyboard and .xib files. It relieves localizers of the need to modify .storyboard and .xib files in Interface Builder. Instead, an app has just one set of .storyboard and .xib files where strings are in the development language—the language that you used to create the .storyboard and .xib files. These .storyboard and .xib files are called the base internationalization. When you export localizations, the development language strings are the source that is translated into multiple languages."

**The key benefits of using base internationalization is we just modify a storyboard for all languages, all localization strings are sperated in different files.**

##ibtools to generate new strings
When you update the UI, such as add view controllers, you need to update the storyboard strings. ibtools is the tool to generate strings from Storyboard. Use the following code to generate a new strings

`ibtool --export-strings-file new_file.strings file.storyboard`

Now, we have two strings files for the same storyboard. One is file.strings, the other is new_file.strings. We also have some other file.strings in different langugages directories. What we should do is finding out the differences between new_file.strings and file.strings and updating the different strings to other language file.strings. If we do this manually, it's very hard to find out the differences. Therefore, we need tools. The Localization Suite(http://www.loc-suite.org/) is a free tool to do such jobs.

You should copy the new_file.strings to the default language folder and replace the old one, then open the Localization suite, you can synchronize the strings, which is very easy.