---
layout: post
title:      "Portfolio Project 1: CLI Data Gem"
date:       2021-06-09 13:27:54 -0400
permalink:  portfolio_project_1_cli_data_gem
---

Flatiron School's first project tasked me with creating a command line interface (CLI) application. It would have to scrape data from a website, define classes that correspond to whatever the data represents, and provide an interface for users to interact with the objects modeling our data. Given the previous lessons, and the abundance of list-based websites on the topic, I made the obvious choice: books.

Goodreads has a few nice book lists with each associated webpage offering a simple layout and links to individual list item's details page. Their *Most Read Books This Week In The United States* felt like a good choice. It's content is dynamic and structure amenable to scraping. I inspected the main list page and a few individual book pages and determined the following attributes, as defined in the class below, would work for this project:

```
class MostReadBooks::Book

  attr_accessor :title, :author, :url, :ratings, :readers, :doc, :format, :page_count, :publisher, :summary, :about_author

  def initialize(title=nil, author=nil, url=nil, ratings=nil, readers=nil)
    @title = title
    @author = author
    @url = url
    @ratings = ratings
    @readers = readers
    self.class.all << self
  end
  . . .
	
end
```

The data stored in the instance variables defined in `initialize` could be scraped directly from the main book list page and the remaining methods defined using `attr_accessor`  would extract their data from individual book pages.

The first method called when app starts up is, appropriately named I think, `start`.

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
  . . .
	
end
```

This method instanitates a new scraper object and and calls the **scrape_books** on this new instance: 
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

It passes that data to the Book class where a new book instance is created.

```
class MostReadBooks::CLI
  . . .
	
  def list_books
    print "How many books would you like to see? Enter a number from 1-#{MostReadBooks::Book.all.length}: "
    @input = gets.strip.to_i
    puts ""
    if (1..MostReadBooks::Book.all.length).include?(@input)
      puts "---Top #{@input} Most Read Books This Week"
      MostReadBooks::Book.print_books(@input)
      puts ""
      get_book
    else
      puts "Please enter a number from 1-#{MostReadBooks::Book.all.length}."
      puts ""
      list_books
    end
  end
  
  def get_book
    print "Select a book number for details (1-#{@input}): "
    book_number = gets.strip.to_i
    if (1..@input).include?(book_number)
      book = MostReadBooks::Book.find(book_number)
      puts ""
      display_book(book)
    else
      puts "Please enter a number from 1 - #{@input}."
      get_book
    end
  end
  
  def display_book(book)
    puts "---Number #{MostReadBooks::Book.all.index(book) + 1} Most Read Book This Week"
    puts "Title: #{book.title}"
    puts "Author: #{book.author}"
    puts "Publisher: #{book.publisher}"
    puts "Format: #{book.format}"
    puts "Page Count: #{book.page_count}"
    puts ""
    puts "#{book.title} has been read by #{book.readers} people this week."
    puts ""
    puts "---Summary"
    puts book.summary
    puts ""
    puts "---About Author"
    puts book.about_author
    puts ""
    see_more_books_or_exit
  end
  . . .
	
end
```

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
	
end
```

```
class MostReadBooks::Book
  . . .
	
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
  . . .
	
end
```

Note that in both the methods definded above doc.css(...) is being passed to format_text. These two attributes presented unique challenges in that on the website they are presented as paragraphs structured to present information in a particular way. 

The format_paragraphs method is something I wrote to maintain the structure of the texts as seen on the website while being presented to the user of Most Read Books in plain text in the command line.  This proved to be tricky as the formatting was not uniform and there were a number of unique edge cases that popped up and had to be dealt. 

I didn't want summary and about_author to hold one large chunk of confusing text, but a string formatted to print to the screen with the exact same structure presented on GoodReads.

rather than capture a large block of text and format it afterwards I thought it best to capture the formatting as it came in.

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
