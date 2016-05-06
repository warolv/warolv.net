---
layout: post
title: "Cache expiration by custom fields"
date: 2016-05-06 15:43
comments: true
categories:  [cache]
description: Rails caching, custom fields caching, activesupport concerns
keywords: rails caching, custom fields caching, memcahed, activesupport concerns
sharing: true
---

<img src="{{ root_url }}/images/memcached.png"/> 

I recently worked on the performance issue, which was solved by caching and I will describe below example of the solution which I came to.

#### Goal
1. Cache objects by custom object's fields.
2. Not use sweeper, same reason as I don't like observers - not obvious enough for developers to understand through models which model uses sweeper or observer.
3. No caching code in models - all caching code must be extracted out of the model, ActiveSupport::Concern helped with that, to be more specific model was extended to use specific caching callback.

<!-- more -->
#### Models and relations

##### business.rb
``` ruby 
class Business < ActiveRecord::Base
  has_many :links
  has_many :documents
  ...

  expire_data fields: [:locale, :business_description, :phone_info]
end
``` 

##### link.rb
``` ruby 
class Link < ActiveRecord::Base
  belongs_to :business
  ...

  expire_data fields: [:url, :display, :link_type]
end
``` 

##### document.rb
``` ruby 
class Document < ActiveRecord::Base
  belongs_to :business
  ...

  expire_data fields: [:file, :description]
end
``` 
  
#### Building expiration module using ActiveSupport::Concern
##### lib/cache/expire_data.rb
``` ruby 
module Cache
  module ExpireData
    extend ActiveSupport::Concern

    module ClassMethods
      def expire_data(*args)
        options = args[0]
        
        expire_on_fields  = options[:fields]
        set_callback :update, :after, -> () {expire_cache(expire_on_fields, expire_block)}
      end
    end

    def expire_cache(expire_on_fields, expire_block)
      if expire_on_fields.count > 0 && (self.changed & expire_on_fields.collect { |f| f.to_s }).any?
        if self.instance_of? Business
          uid = self.uid
        else
          uid = self.business.uid
        end

        Rails.cache.delete("data/#{uid}")
      end
    end
  end
end
``` 

#### Adding concern to initializers
##### config/initializers/extensions.rb:
``` ruby 
ActiveRecord::Base.send(:include, Cache::ExpireData)
``` 

#### Fetching cached data:
``` ruby 
business_uid = Business.find(params[:business_id]).uid
cached_data = Rails.cache.fetch("data/#{business_uid}) {/Fetch fresh data if cache is expired/}
...
``` 