---
layout: post
title: "Migrating Facebook apps to Open Graph 2.0"
date: 2015-04-19 13:11
comments: true
categories:  [facebook, open graph, migrate web apps, facebook connect]
description: Upgrade facebook apps, migrate to open graph 2.0, omniauth facebook gem, facebook connect gem
keywords: facebook upgrade apps, upgrade open graph, migrate facebook apps, facebook javascript SDK
sharing: true
---

Lately I got a message from Facebook about upgrading of our Facebook apps to OpenGraph2:

<img src="{{ root_url }}/images/facebook-warning.png" />

<!-- more -->

We have 3 apps - 2 web applications which used Facebook Javascript SDK and Facebook connect app. Let's see what's need to be done to upgrade web apps to latest Open Graph and javascript SDK, also forgot to say That I don't need to submit to login review, cause apps using standard permissions - public_profile, email, user_friends.

### Update web apps

Updating reference to Facebook javascript SDK:

before:
``` ruby 
  '//connect.facebook.net/en_US/all.js'
``` 

after:
``` ruby 
  '//connect.facebook.net/en_US/sdk.js'
``` 
Add version 2.0 explicitly:

before:
``` ruby 
  FB.init({appId: app_id, status: true, cookie: true});
``` 

after:
``` ruby
  FB.init({appId: app_id, status: true, cookie: true, version: 'v2.0'});
``` 

### Update facebook connect
For facebook connect we using ['omniauth-facebook'](https://github.com/mkdynamic/omniauth-facebook) gem.
To update gem to latest open graph version you need to make changes in your configuration by adding
:site => 'facebook/opengraph2/2.0' to 'client_options'.

### config/initializers/omniauth.rb:
``` ruby
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :facebook, 'facebook_login', 'facebook_secret', 
    {:client_options => {:site => 'https://graph.facebook.com/v2.0'}}
end
``` 

You may test all changes in your app before deploying to production by turning all flags on in settings/migration tab.

<img src="{{ root_url }}/images/facebook-migrations.png" />
