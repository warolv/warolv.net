---
layout: post
title: "Wordpress single page analysis or passive information gathering"
date: 2017-05-22 12:24
comments: true
categories: [wordpress hacking, web security]
description: wordpress passive information gathering, wordpress single page analysis
keywords: wordpress info gathering, wordpress information gathering, wordpress passive information gathering, wordpress hacking, wordpress single page analysis
sharing: true
draft: false
---
<img src="{{ root_url }}/images/wordpress-analysis.png" align="right"/> 

#### All code and information for educational purposes only

Today I will show you how you can gather information about WordPress by loading and analyzing a single page; and then, based on this knowledge we will create script to automate the entire process.  
Why analyze only one page? Why not use existing automated solutions like [nikto](https://cirt.net/nikto2) or [WPScan](https://wpscan.org/)?  
Those are great tools with plenty of functionality, but they create a lot of noise because they require multiple requests for targeting the site. That may be a problem, because if you need to perform reconnaissance on a WordPress site and remain undetected, your behavior must be the same as a regular user trying to read a blog on the WordPress site.

#### Discover wordpress version ####
First, let’s try to find the WordPress version of a target. You can do this by going to the WordPress main site, clicking “View Page Source” in a Chrome browser, and finding the meta tag  named “generator.”

``` html  
<meta name="generator" content="WordPress 4.7.5" />
``` 
You can see wordpress version inside content attribute.

<!-- more -->

Let's get wordpress version with ruby:
``` ruby   
  require 'nokogiri'
  require 'net/http'
  require 'open-uri'

  url = 'http://www.example.com' # Put url of your site here
  doc = Nokogiri::HTML(open(url).read)
  doc.xpath("//meta[@name='generator']/@content").each do |attr|
    puts attr.value
  end
``` 

Check out my [wp_info_gathering.rb](https://github.com/warolv/wordpress-scripts/blob/master/wp_info_gathering.rb) script which already do this.

To remove the meta=“generator” tag, add this line to your functions.php:
remove_action(‘wp_head’, ‘wp_generator’);

#### Discover xml-rpc protocol location ####
The XML-RPC is an API that enables developers to create WordPress “apps” (like clients, plugins and themes), which allow you to make remote HTTP requests to your WordPress site ([link to full article](https://blogvault.net/how-xml-rpc-affects-wordpress-security/)).
You may use a brute-force password via XML-RPC. I described this process and created script for this purpose;  [read full article](http://warolv.net/blog/2017/05/14/wordpress-brute-force-password-attack-using-xmlrpc-api).

To find the location of XML-RPC, look for two link tags with attributes “pingback” and “EditURI” accordingly:
``` html  
<link rel="pingback" href="http://www.example.com/xmlrpc.php" />
....
<link rel="EditURI" type="application/rsd+xml" title="RSD" href="http://www.example.com/xmlrpc.php?rsd" />
``` 

Let's find links with ruby:
``` ruby   
  doc.xpath("//link[@rel='pingback']/@href").each do |attr|
    puts attr.value
  end

  doc.xpath("//link[@rel='EditURI']/@href").each do |attr|
    puts attr.value
  end
``` 

[How to remove those tags from your html](http://infoheap.com/remove-xmlrpc-from-wordpress-headers/)

#### Discover wordpress REST API ####
The WordPress REST API provides API endpoints for WordPress data types that allow developers to interact with sites remotely by sending and receiving JSON (JavaScript Object Notation), enabled by default starting from version 4.7 ([link to full article](https://developer.wordpress.org/rest-api/)). 

Using REST API, we can see all the WordPress users, and then, using a brute-force attack via XML-RPC, we can find out the password.
The combination of two enabled APIs (XML-RPC and REST API) may be very dangerous, so you should disable XML-RPC if it’s not in use.

As an illustration, you can get user info by entering in your browser: http://www.example.com/wp-json/wp/v2/users   
The results may look like this:

``` json
[{"id":1,"name":"admin","url":"","description":"","link":"http:\/\/www.example.com\/author\/admin\/","slug":"admin","avatar_urls":{"24":"http:\/\/0.gravatar.com\/avatar\/fa678936f8098020aafd0e0e9b91ec5d?s=24&d=mm&r=g","48":"http:\/\/0.gravatar.com\/avatar\/fa678936f8098020aafd0e0e9b91ec5d?s=48&d=mm&r=g","96":"http:\/\/0.gravatar.com\/avatar\/fa678936f8098020aafd0e0e9b91ec5d?s=96&d=mm&r=g"},"meta":[],"_links":{"self":[{"href":"http:\/\/www.example.com\/wp-json\/wp\/v2\/users\/1"}],"collection":[{"href":"http:\/\/www.example.com\/wp-json\/wp\/v2\/users"}]}}]
``` 
You can see the user, “admin” in this case; and there you have valuable information for a brute-force attack.

I will describe user enumeration and obtaining a users list through REST API in my next post.

This link tag indicates that the WordPress site is using REST API:

``` html  
<link rel='https://api.w.org/' href='http://www.example.com/wp-json/'/>
``` 

Let's find this link with ruby:
``` ruby   
  doc.xpath("//link[@rel='https://api.w.org/']/@href").each do |attr|
    puts attr.value
  end
``` 

If this tag exist in html, REST API enabled. 

[How to Disable XML-RPC in WordPress](http://www.wpbeginner.com/plugins/how-to-disable-xml-rpc-in-wordpress/)


#### Discover wordpress plugins ####
To find reference to plugins in HTML, look for link tags and script tags that look like: href=‘http://www.example.com/wp-content/plugins/...

In my case it's:
``` html  
<link rel='stylesheet' id='rs-plugin-captions-css'  href='http://www.example.com/wp-content/plugins/revslider/rs-plugin/css/captions.php?rev=4.3.8&#038;ver=4.7.5' type='text/css' media='all' />
<link rel='stylesheet' id='inbound-shortcodes-css'  href='http://www.example.com/wp-content/plugins/cta/shared/shortcodes/css/frontend-render.css?ver=4.7.5' type='text/css' media='all' />
<link rel='stylesheet' id='dry_awp_theme_style-css'  href='http://www.example.com/wp-content/plugins/advanced-wp-columns/assets/css/awp-columns.css?ver=4.7.5' type='text/css' media='all' />
<link rel='stylesheet' id='easy_table_style-css'  href='http://www.example.com/wp-content/plugins/easy-table/themes/default/style.css?ver=1.5.2' type='text/css' media='all' />
...
<script type='text/javascript' src='http://www.example.com/wp-content/plugins/cta/shared/assets/frontend/js/analytics/inboundAnalytics.js?ver=4.7.5'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/plugins/revslider/rs-plugin/js/jquery.themepunch.plugins.min.js?rev=4.3.8&#038;ver=4.7.5'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/plugins/revslider/rs-plugin/js/jquery.themepunch.revolution.min.js?rev=4.3.8&#038;ver=4.7.5'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/plugins/cta/js/cta-load-variation.js?ver=1'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/plugins/cta/shared/assets/global/js/jquery.cookie.js?ver=4.7.5'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/plugins/cta/shared/assets/global/js/jquery.total-storage.min.js?ver=4.7.5'></script>
``` 

Extracted plugins:  

* revslider
* cta
* advanced-wp-columns
* easy-table

Knowing the plugins used, you can find ways to exploit.

Let's extract plugins with ruby:
``` ruby   
  all_links = doc.xpath('//@href | //@src').map(&:value)
  plugin_links = links.find_all{|l| l.include?('/wp-content/plugins/')}

  plugins = []
  plugin_links.each do |link|
    link_parts = link.split('/')
    plugins << link_parts[link_parts.index('plugins') + 1]
  end

  p plugins.uniq
``` 

#### Discover wordpress themes ####
To find reference on to themes in html look for link tags and script tags which looks like: href='http://www.example.com/wp-content/themes/....

In my case it's:
``` html  
<script src="http://www.example.com/wp-content/themes/interface-pro/js/html5.js"></script>
<link rel='stylesheet' id='interface_style-css'  href='http://www.example.com/wp-content/themes/interface-pro/style.css?ver=4.7.5' type='text/css' media='all' />
<link rel='stylesheet' id='interface-responsive-css'  href='http://www.example.com/wp-content/themes/interface-pro/css/responsive.css?ver=4.7.5' type='text/css' media='all' />
<link rel='stylesheet' id='jquery_fancybox_style-css'  href='http://www.example.com/wp-content/themes/interface-pro/css/jquery.fancybox-1.3.4.css?ver=4.7.5' type='text/css' media='all' />
<link rel='stylesheet' id='easy_table_style-css'  href='http://www.example.com/wp-content/plugins/easy-table/themes/default/style.css?ver=1.5.2' type='text/css' media='all' />
...
<script type='text/javascript' src='http://www.example.com/wp-content/themes/interface-pro/js/backtotop.js?ver=4.7.5'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/themes/interface-pro/js/scripts.js?ver=4.7.5'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/themes/interface-pro/js/jquery.custom.js?ver=4.7.5'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/themes/interface-pro/js/jquery.easing.1.3.js?ver=1'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/themes/interface-pro/js/jquery.quicksand.js?ver=1'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/themes/interface-pro/js/jquery.cycle.all.min.js?ver=2.9999.5'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/themes/interface-pro/js/jquery.fancybox-1.3.4.pack.js?ver=1.3.4'></script>
<script type='text/javascript' src='http://www.example.com/wp-content/themes/interface-pro/js/interface-custom-fancybox-script.js?ver=4.7.5'></script>
``` 

Extracted themes:

* interface-pro

Knowing which themes are used, you can look for ways to exploit.

Let's do it with ruby:
``` ruby   
  all_links = doc.xpath('//@href | //@src').map(&:value)
  theme_links = links.find_all{|l| l.include?('/wp-content/themes/')}

  themes = []
  theme_links.each do |link|
    link_parts = link.split('/')
    themes << link_parts[link_parts.index('themes') + 1]
  end

  p themes.uniq
``` 

#### Conlusion ####
Of course, we may gather even more info by additional requests; for example, we may download and analyze “/readme.html” to get the version of WordPress used.  
We can also get more information about the plugins and themes by analyzing downloaded JS files and CSS files, as well as by analyzing “http” headers.
But the purpose of this post is to show that you can get a lot of information about WordPress from just a single-page analysis.

Thank you for reading my post. Soon, I plan to create script that will automate this entire process, and you will be able to [download](https://github.com/warolv/wordpress-scripts/blob/master/wp_single_page_analysis.rb) and use it.