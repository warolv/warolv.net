---
layout: post
title: "Customizing 'save changes' alert behaviour"
date: 2013-11-09 16:02
comments: true
categories: [alert behaviour, save changes alert]
description: Cutomize alert behaviour, triggering save changes, leave a page alert behaviour
keywords: alert behaviour cutomization, triggering save changes, leave page alert behaviour
sharing: true
---

In the current project I am working on, there was a need to trigger a save changes alert on a group of different pages in case some changes were made on the page, for things like entering data to inputs or checking some checkboxes. The problem was that on certain pages, and sometimes on certain parts of specific pages, I needed to disable this behaviour.

<!-- more -->

<img src="{{ root_url }}/images/save-changes2.png" />

The solution I came up with:

``` coffeescript 
	->
	  changed = false
	  notify_before_close = true

	  $(window).bind 'beforeunload', ->
	    'There are unsaved changes.' if changed and notify_before_close
	  
	  $("input[type=text], input[type=radio], input[type=hidden], 
	  input[type=color], textarea, select, input[type=checkbox]").change ->
        if (!$(this).hasClass('discard-changes') 
        && !$(this).parents().hasClass('discard-changes'))
          changed = true
``` 
 1. If some things were changed on the page, we will mark the changed parameter as true.<br>
 2. If we don’t want to use this behaviour on a specific page, we will mark notify_before_close parameter as false.<br>
 3. If we want to disable this behaviour for a specific group of inputs, we need to wrap these inputs with a specific class – ‘discard-changes’. For example:

``` haml 
  = form_for @post, :html => {:multipart => true, :class => 'ajax-form'} do |f|
	  .field
	    = f.label :tags
	    = f.text_field :tags
	  .field.discard-changes
	    = f.label :name
	    = f.text_field :name
	    = f.label :description
	    = f.text_field :description
	  .actions
	    = f.submit
```
In this case, changes to name and description input will not trigger a save changes alert. I hope this helps someone out there. 
