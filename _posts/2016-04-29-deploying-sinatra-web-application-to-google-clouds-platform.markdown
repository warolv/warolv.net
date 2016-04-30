---
layout: post
title: "Deploying sinatra web application to google clouds platform"
date: 2016-04-29 12:54
comments: true
categories:  [google cloud platform, sinatra]
description: Google cloud platform deployment, sinatra application deployment, google cloud SDK, gcloud CLI, google SDK for ruby 
keywords: google cloud platform, google SDK, sinatra, gcloud, sinatra, google SDK, google SDK ruby
sharing: true
---

### Creating basic 'hello world' app with sinatra

#### Application structure
```
app/main.rb
config.ru
Gemfile
Rakefile
``` 

#### main.rb
``` ruby 
get '/' do
  "Hello World!"
end
``` 
<!-- more -->

#### config.ru
``` ruby 
  require './app/main'
  run Sinatra::Application
``` 

####  Test everything works properly
1. Run bundle exec rackup -p 7000 config.ru
2. In browser localhost:7000 must return 'hello world' string

### Deploying app to google clouds

#### Create a project in the Google Cloud Platform Console
You may find exact instructions [here](https://cloud.google.com/ruby/getting-started/hello-world).
Projects enable you to manage all Google Cloud Platform resources for your app,including deployment, access control, billing, and services.

1. Open the Cloud Platform Console. (https://console.cloud.google.com)
2. In the drop-down menu at the top, select Create a project.
3. Click Show advanced options. Under App Engine location, select a United States location.
4. Give your project a name.
5. Make a note of the project ID, which might be different from the project name. The project ID is used in commands and in configurations.

#### Enable billing for your project.
If you haven't already enabled billing for your project, enable billing now. Enabling billing allows the application to consume billable resources such as running instances and storing data.
(https://console.cloud.google.com/project/_/settings)

#### Install the Google Cloud SDK.
* Go to https://cloud.google.com/sdk/#mac (in may case it's for mac), download archive, extract files to folder
* Copy folder to appropriate locataion, in my case: cp -R google-cloud-sdk /Users/'You username'/Library/Application Support/google-cloud-sdk
* cd  /Users/'You username'/Library/Application Support/google-cloud-sdk and run ./install.sh

#### Add app.yaml to your app
``` ruby 
runtime: ruby
vm: true
entrypoint: bundle exec rackup -p 8080 -E production config.ru
``` 

#### Deploy of app to google clouds platform:
* In your app's folder run 'gcloud preview app deploy' from console
* You url must look like https://<your-project-id>.appspot.com, in my case it's: hello-world-1291.appspot.com

### Misc Tips and Tricks

#### Debugging your app with 'pry'

* Add to main.rb: require 'pry'

``` ruby 
group :development, :test do
  gem 'pry'
  gem 'pry-remote'
  gem 'pry-nav'
end
``` 

#### Return response in json:
#### main.rb
``` ruby 
require 'json'

get '/example.json' do
  content_type :json
  { :key1 => 'value1', :key2 => 'value2' }.to_json
end
``` 
