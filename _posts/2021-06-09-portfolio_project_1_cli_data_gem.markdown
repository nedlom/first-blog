---
layout: post
title:      "Portfolio Project 1: CLI Data Gem"
date:       2021-06-09 13:27:54 -0400
permalink:  portfolio_project_1_cli_data_gem
---


Tasked with building a command line interface (CLI) that scrapes data from a website, has classes that define and create objects that model the data, and offers a user-friendy, text-based interface to interact with those objects, I selected a website that offered a list-based presentation of items that would be familiar enough to work with given the previous lessons in the course: books. 

Goodreads offers a few "most read" book lists. I chose the Most Read Books This Week In The United States list.

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

The instance variables set in #initialize represent data that can be scraped directly from the main page and the remaining attr_accessors tell us everything that can be retrieved from a book's personal webpage. 

The first function called upon is #start.

```
class MostReadBooks::CLI

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

This is the only method defined in the Scraper class. It's job is to pass the webpage's HTML to Nokogiri's Nokogiri::HTML method which creates a Nokogiri object (NodeSet) that we call #css method on to extract data. It then iterates over a NodeSet and uses the #css method to grab the ...


```
class MostReadBooks::Book
  . . .
	
  def doc
    @doc ||= Nokogiri::HTML(open(self.url).read)
  end
  
  def format
    @format ||= doc.css("#details .row")[0].text.split(/, | pages/).first
  end
	. . .
	
```

```
  def summary
    @summary ||= format_text(doc.css("#description span").last) #[1]
  end
  
  def about_author
    @about_author ||= if doc.css(".bookAuthorProfile span").empty?
      @about_author = "There is no information for this author"
    else
      if doc.css(".bookAuthorProfile span").length == 2
        @about_author = format_text(doc.css(".bookAuthorProfile span").last) #[1]
      else
        @about_author = format_text(doc.css(".bookAuthorProfile span").first) #[0]
      end
    end
  end
```

Note that in both the methods definded above doc.css(...) is being passed to format_text. These two attributes presented unique challenges in that on the website they are presented as paragraphs structured to present information in a particular way. 

The format_paragraphs method is something I wrote to maintain the structure of the texts as seen on the website while being presented to the user of Most Read Books in plain text in the command line.  This proved to be tricky as the formatting was not uniform and there were a number of unique edge cases that popped up and had to be dealt. 

```
  def format_text(element)
    # issues at: 1, 8, 17, 21, 32, 50
    # else condtions deals with non-text nodes that have their own text/formatting
    text_array = element.children.map do |node|
      if node.children.empty? 
        node.text
      else
        node.children.map do |a|
          a.text
        end
      end
    end.flatten
    
    text_groups = text_array.chunk do |line|
      line != "" && line != " "
    end.to_a
    
    paragraphs = text_groups.map do |group|
      group[1].join if group[0]
    end.compact
    
    # removes a reference to an image seen on webpage.
    paragraphs.delete_at(0) if paragraphs[0].downcase.include?("edition")

    paragraph_lines = paragraphs.map do |paragraph|
      paragraph.scan(/(.{1,75})(?:\s|$)/m)
    end

    paragraph_lines.map do |line|
      line.join("\n") 
    end.join("\n\n")
  end
```

Don't get hung up on trying to build something profound.
My suggestion is to not spend too much time on finding the perfect website or coming up with the most interesting idea. Pick a website that approximates the examples highlighted in the lessons and tutorial and just start coding. What makes a website harder or easier to work with will become clear as you start going through of building your application. At this point it's not such a big deal to scrap what you've been doing and choose another website because the way forward and potential pitfalls will be clear.  
