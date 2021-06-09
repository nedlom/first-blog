---
layout: post
title:      "Portfolio Project 1: CLI Data Gem"
date:       2021-06-09 13:27:54 -0400
permalink:  portfolio_project_1_cli_data_gem
---


Tasked with building a command line interface (CLI) that scrapes data from a website, has classes that define and create objects that model the data, and offers a user-friendy, text-based interface to interact with those objects, I selected a website that offered a list-based presentation of items that would be familiar enough to work with given the previous lessons in the course: books. 

Goodreads offers a number of "most read" book lists. I chose the most read books in United States this week.

Most Read Books This Week In The United States

Each book listed on the main page is linked to a seperate page that provides extra details. A quick scan of both these sources implied that the following attributes could be retrieved:

```
  attr_accessor :title, :author, :url, :ratings, :readers, :doc, :format, :page_count, :publisher, :summary, :about_author

  def initialize(title=nil, author=nil, url=nil, ratings=nil, readers=nil)
    @title = title
    @author = author
    @url = url
    @ratings = ratings
    @readers = readers
    self.class.all << self
  end
```

The instance variables defined in the initialize represent the data that can be scraped from the main page and the remaining attr_accessors tell us everything that can be retrieved from a book's personal webpage. 

The first function called upon is #start.

```
  def start
    MostReadBooks::Scraper.new.scrape_books
    puts ""
    puts "#{"-" * 30}Most Read Books#{"-" * 30}"
    puts "Welcome to Most Read Books. This application showcases the #{MostReadBooks::Book.all.length} most read"
    puts "books in the United States this week (according to Goodreads)." 
    puts ""
    list_books
  end
```

This method intializes a new scraper and and calls the #scrape_books on this new instance: 
```
  
class MostReadBooks::Scraper
  def scrape_books
    page = Nokogiri::HTML(open("https://www.goodreads.com/book/most_read").read)
    page.css("tr").each do |b|
      title = b.css("[itemprop='name']")[0].text
      author = b.css("[itemprop='name']")[1].text
      url = "https://www.goodreads.com/#{b.css(".bookTitle")[0]['href']}"
      ratings = b.css(".minirating").text.strip
      readers = b.css(".greyText.statistic").text.strip.split("\n")[0]
      MostReadBooks::Book.new(title, author, url, ratings, readers)
    end
  end
end
```

This is the only method defined in the Scraper class. It's job is to pass the webpage's HTML to Nokogiri's Nokogiri::HTML method which creates a Nokogiri object (NodeSet) we can call methods on that allow us to easily extract the desired data. It then iterates over a NodeSet and uses the #css method to grab the ...


Don't get hung up on trying to build something profound.
My suggestion is to not spend too much time on finding the perfect website or coming up with the most interesting idea. Pick a website that approximates the examples highlighted in the lessons and tutorial and just start coding. What makes a website harder or easier to work with will become clear as you start going through of building your application. At this point it's not such a big deal to scrap what you've been doing and choose another website because the way forward and potential pitfalls will be clear.  
