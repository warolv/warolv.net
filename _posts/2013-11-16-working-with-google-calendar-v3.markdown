---
layout: post
title: "Connecting your app to google calendar v3 via oauth2"
date: 2013-11-16 15:49
comments: true
categories: [google calendar API, oauth2]
description: Upgrade google calendar, ruby google API client, ruby google calendar gem, google refresh token, getting busy time from google calendar
keywords: google calendar API, ruby gem google calendar, google API refresh token, google calendar busy times
sharing: true
---

### Register your app with the google API console ###
  1. Go to https://code.google.com/apis/console and register your app.<br>
  2. Enable google calendar(ON) in services section.<br>
  3. Get CLIENT ID and CLIENT SECRET from API Access.<br>
  4. Fill callback urls - click API Access -> client ID for web applications section -> edit settings and fill it with your callback url - for example http://localhost/auth/google_oauth2/callback for local debugging and one for production server. Also fill Authorized JavaScript Origin - http://localhost for example. If you have path starting with https add this in separate string also.
  
  Be carefull - this step most important, cause you may spend a lot of time figuring out what problem is if you not created your account correctly and not filled callback
  urls, that what happend in my case:-). 

<!-- more -->

### Adding necessary gems to your gem file ###

``` ruby 
  gem 'google-api-client', :require => 'google/api_client' 
  gem 'omniauth-google-oauth2', 
  :git => 'https://github.com/zquestz/omniauth-google-oauth2.git'
``` 
 
 
 First gem is google calendar v3 api client which interacts with google services, second one implements oauth2 authentication to work with google v3 services.

 bundle install

### Adding your client id and client secret to app/initializers/omniauth.rb for oauth2 authentication ###

``` ruby 
Rails.application.config.middleware.use OmniAuth::Builder do
 provider :google_oauth2, , 'oauth2_client_id', 'oauth2_client_secret' {
   access_type: 'offline',
   approval_prompt: 'force',
   scope: 'https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/calendar',
 }
``` 

### Getting and saving oauth2 google token to access a google calendar API ###
1. Enter omniauth path in browser - '/auth/google_oauth2 which' -  must be available after you installed omniauth-google-oauth2 gem.
2. Press allow access, if you not registered callback in Google API console you will get error on this stage.
3. After finishing you will be redirected to your callback( "/auth/:provider/callback") path - in our case it's -  "/auth/google_oauth2/callback".
4. Add route to routes.rb - something like "/auth/:provider/callback" => "googleauth#create".

``` ruby 
class GoogleOuath2Controller < ApplicationController   
  def create     
    @auth = request.env["omniauth.auth"]
    # need this token to work with google services
    @token = @auth["credentials"]["token"]
    @refresh_token = @auth["credentials"]["refresh_token"]
    @expires_at = @auth["credentials"]["expires_at"]
          
    # Save credentials in DB, session or yaml file - in my case i am saving 
    # in auth table, it may be comfortable to save all omniauth tokens 
    # in one table if you use many providers like facebook, twitter... 
    Auth.create(:provider => google_oauth2, :token => @token, 
    :refresh_token => @refresh_token, :expires_at => @expires_at, ...) 
  end
end
``` 

  Also it's highly reccomended to save refresh_token and expires_at - you may be surprized to know that after some time (in my case it was about an hour) you 
  token expires, you need refresh token to get new token after expiration and expire_at to know how much time you 
  have for this token.

### Connect to google calendar and get all calendars list ###


``` ruby 
def init_client
  client = Google::APIClient.new
  # Fill client with all needed data
  client.authorization.access_token = @token #token is taken from auth table
  client.authorization.client_id = @oauth2_key
  client.authorization.client_secret = @oauth2_secret
  client.authorization.refresh_token = @refresh_token
  return client
end

def get_all_calendars
  client = init_client

  service = client.discovered_api('calendar', 'v3')
  @result = client.execute(
    :api_method => service.calendar_list.list,
    :parameters => {},
    :headers => {'Content-Type' => 'application/json'})
end
```

### Extracting busy times ###
    
Get all the busy times currently available for the given token
 
``` ruby 
def get_busy_times
  client = init_client
  service = client.discovered_api('calendar', 'v3')
  @result = client.execute(
    :api_method => service.freebusy.query,
    :body_object => { :timeMin => start_time, #example: DateTime.now - 1.month
                      :timeMax => end_time, #example: DateTime.now + 1.month
                      :items => items
                    },
    :headers => {'Content-Type' => 'application/json'}})
end
```

### Getting new token if old one is expired ###

``` ruby 
def execute_and_update_token(client, options)
  client = init_client
  old_token = client.authorization.access_token

  service = client.discovered_api('calendar', 'v3')
  @result = client.execute(
    :api_method => service.calendar_list.list,
    :parameters => {},
    :headers => {'Content-Type' => 'application/json'})

  result = client.execute(options)
  
  new_token = client.authorization.access_token
  if old_token != new_token
    Auth.update_attribute(:token => new_token)
  end
end
```

Thats all, thank you:-).
