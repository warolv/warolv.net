---
layout: post
title: "Dynamically marking fields with error in rails form"
date: 2013-11-03 21:29
comments: true
sharing: true
categories: [rails form errors, dynamic error handling, marking form fields dynamically]
description: Dynamically mark rails form errors, custom rails form errors dynamically, rails errors using ajax, mark form fields with errors via ajax in rails.
keywords: dynamically marking fields, rails form errors, form errors ajax, rails dynamic errors, mark form fields
sharing: true
---

Recently i had to deal with a problem of dynamically marking form fields with error and displaying specific error for this field below, in case of validation error. After trying to find alredy existing solution and 2 hours spend on stackoverflow.com looking for answer with no success i started to build my own solution.

In my example, I will create a blog post dynamically by using ajax to fill a form with the needed data; submitting it must return status code :ok(200) and create a post. In case you have any validation errors, the corresponding inputs must be marked with an error and the text of the error must be put below the input field. Take a look at this picture to better understand what I mean:

<!-- more -->

<img src="{{ root_url }}/images/post_form_example.png" />

Let's make it:

posts_controller.rb:

``` ruby 
	class PostsController < ApplicationController
	  def update
	    @post.attributes = params[:post]
	    if @post.save
	      head :ok
	    else
	      render :json => @post.errors, :status => :unauthorized
	    end
	  end
	end
``` 

post_form_handlers.js.coffee

I using rails ujs for events binding.

``` coffeescript 
	$(document).on 'ajax:beforeSend', '.ajax-form', () ->
	  $('#form-errors').empty()
	  clearErrors()

	$(document).on 'ajax:success', '.ajax-form', (data, status, xhr) ->
	  $("#form-errors").html "<div class='success'>Post update success</div>"
	  clearErrors()

	$(document).on 'ajax:error', '.ajax-form', (data, status, xhr) ->
	  $("#form-errors").html "<div class='error'>Post update error</div>"
	  markFormErrors(status, false)

	window.markFormErrors = (status) ->
	  try
	    errors_array = JSON.parse(status.responseText)
	    for key of errors_array
	      selector = '[id$=' + key + ']'
	      if $(selector).length > 0 && $(selector).data('errorOn') != undefined
	        markWithError(selector, errors_array[key])
	      else
	        $('#form-errors').append "<div class='error'>"+key+': '+errors_array[key]+"</div>"
	  catch

	window.markWithError = (field_selector, error) ->
	  $(field_selector).after "<div class='formError'>" + error + "</div>"
	  row.find("label:first").wrap "<div class='field_with_errors'></div>"
	  $(field_selector).wrap "<div class='field_with_errors'></div>"

	window.clearErrors = () ->
	  $('input').each ->
	    row = $(this).parents('.field')
	    row.find('.formError').remove()
	    row.find('.field_with_errors').find('>input,>label').unwrap()
```       

new.html.haml
``` haml 
	- content_for :javascripts do
      = javascript_include_tag 'post_form_handlers'

	#form-errors

	= form_for @post, :remote => true, :html => {:multipart => true, :class => 'ajax-form'} do |f|
	  .field
	    = f.label :name
	    = f.text_field :name, :data => {'error-on' => true}
	 	.field
	    = f.label :description
	    = f.text_field :description, :data => {'error-on' => true}
	  .field
	    = f.label :tags
	    = f.text_field :tags, :data => {'error-on' => true}
	  .actions
	    = f.submit
```  


post.scss for example

``` scss
.field_with_errors {
  input[type=text], input[type=password], input[type=email], select, textarea {
    border: 1px solid #d3211d;
  }
}

.formError {
	color: red;
}
```  

Now, if you have any validation errors, for example “field can’t be blank” or a length limitation for the specific field, you will see it marked with an error, like in the picture above. Also, you can use the data attribute – error_on to enable or disable this field to be marked with an error. If you have some custom validators in the post model that doesn’t corresponds with any of the fields in the form, it will be automatically added to general errors. #form-errors

