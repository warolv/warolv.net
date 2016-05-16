---
layout: post
title: "Preventing lock of table during migration using LHM (Large Hadron Migrator)"
date: 2016-05-16 08:37
comments: true
categories:  [cache]
description: Rails caching, custom fields caching, activesupport concerns
keywords: rails caching, custom fields caching, memcahed, activesupport concerns
sharing: true 
---
#### Problem definition
Running migration which adds column to a table which has a couple million of records may be problematic, because of lock of this table during migration.
Let's say it not only may be problematic - it was problematic, because of this problem our server was down for a couple of minutes and SQL server was restarted,
so to solve this problem we using 'LHM' gem - Large Hadron Migrator

##### Shortly describing how lhm works
1. Creates new table called lhmn_posts (I am altering posts table in my example) with new column you adding in migration
2. Copies all data from posts table to lhmn_posts.
3. Rename of lhmn_posts to posts and old posts table becomes lhmn_[date]_posts table

<!-- more -->

#### Migration which adds 'version' column to posts table which have 6 millions of records

##### db/add_version_to_posts.rb
``` ruby 
require 'lhm'

class AddVersionToPosts < ActiveRecord::Migration
  def up
    Lhm.change_table :posts do |m|
      m.add_column :version, "INT(12)"
    end
  end

  def down
    Lhm.change_table :posts do |m|
      m.remove_column :version
    end
  end
end
``` 

##### Problem during migration on production you may encounter
1. 'You do not have the SUPER privilege and binary logging is enabled' exception during migration to production, to solve this problem
you need to set specific flags to your SQL db(Mysql in my case):
mysql -u USERNAME -p
set global log_bin_trust_function_creators=1; 
2. After running you migration and getting exception you may need clean temp tables lhm already created like lhmn_posts - to solve this issue just add  Lhm.cleanup(:run) before migration:
``` ruby 
require 'lhm'

class AddVersionToPosts < ActiveRecord::Migration
  Lhm.cleanup(:run)
  
  def up
    Lhm.cleanup(:run)
    ...
  end
end
``` 
3. After migration is finished lhm keeps old table as lhmn_'date'_posts and it's up to you decide when to delete this table.
