---
layout: post
title:      "Portfolio Project 1: CLI Data Gem"
date:       2021-06-09 13:27:54 -0400
permalink:  portfolio_project_1_cli_data_gem
---

Flatiron School's first project tasked me with creating a command line interface (CLI) application. It would have to scrape data from a website, define classes that correspond to whatever the data represents, and provide an interface for users to interact with the objects modeling our data. Given the previous lessons, and the abundance of list-based websites on the topic, I made the obvious choice: books.

Goodreads has a few nice book lists with associated webpages offering simple layouts and links to individual book pages. Their *Most Read Books This Week In The United States* felt like a good choice. It's content is dynamic and structure amenable to scraping. Three classes would suffice: Books, Scraper, and CLI. Inspection of the site determined the attributes, seen defined in the class below, that would be used to describe Book objects:
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
The data stored in instance variables upon initialization, defined in `initialize`, is scraped directly from the *Most Read Books This Week In The United States* list. The remaining methods defined using `attr_accessor` extract their data from individual book pages.

The CLI class will control the action of the program. Upon starting the application a new CLI object is instantiated and calls, the appropriately named method, `start`:  
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
`start` instanitates a Scraper object and and calls `scrape_books` on this new instance: 
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
`scrape_books`, the only method defined in the Scraper class, passes the webpage's HTML to `Nokogiri::HTML` which creates a Nokogiri object that we can call `css` on to extract data. Iterating over a NodeSet we call `css` on each node to grab and store data corresponding to Book attributes in local variables. We then pass these variables to Book's `new` method instaniating and saving a new Book object for each book in the webpage's list. Our CLI class then lists the number of book objects our user wants to see:
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
```
Once our user selects an appropriate number of books to list, `get_book` is called:
```
class MostReadBooks::Book
  . . .
	
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
  . . .
	
end
```
The only instance variable defined in the CLI class is `@input` which shows up in `list_books` and `get_book`. I wanted these methods seperate, but given that a user can select how many books to list, and I only want them to be able to select a book from the list that was printed to the screen, `get_book` depends on the knowledge of how many books have been listed, which is precisely the information `@input` holds, so an instance variable made sense here.

Once the user has selected a book, `display_book` is called:
```
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
Checking the Book class we can see how the attributes used above that were not defined in Book's `initialize` method are obtained:
```
class MostReadBooks::Book
  . . .
	
  def doc
    @doc ||= Nokogiri::HTML(open(url).read)
  end
  
  def format
    @format ||= doc.css("#details .row")[0].text.split(/, | pages/).first
  end
	
	def page_count
    @page_count ||= doc.css("#details .row")[0].text.split(/, | pages/).last
  end
  . . .
	
end
```
`doc` sets (or returns) an instance variable `@doc` which stores a Nokogiri object corresponding to the HTML of a book's individual webpage. `doc` is then used, in conjunction with `css`, in each subsequent method setting an instance variable to the appropriate value scraped from the book's page. So, the instance variables for a book instance that are not initialized during instantiation are only set once a user selects that particular book.

Two methods from the Book class are worth noting:
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
        @about_author = format_text(doc.css(".bookAuthorProfile span").last)
      else
        @about_author = format_text(doc.css(".bookAuthorProfile span").first)
      end
    end
  end
  . . .
	
end
```
Both methods definded above pass `doc.css(...)` into `format_text`. The book summary and about author sections of a book's webpage are given in paragraphs structured to present the text in a certain way, with a particular formatting and flow to the information. Rather than just call `text` on `doc.css(...)` to obtain a large string of text, and try to apply my own formatting to it later when it's being printed to the screen, I decided to build `format_text` to not just get the text from the website, but to also capture the formating used in the webpage's HTML to define the structure of the text. This way we just need to call `puts` on the `summary` or `about_author` methods to achieve a layout almost identical to that viewed in the browser on GoodReads. This was tricky as the formatting was not uniform among book pages, so there were a number of unique cases that had to be considered. To keep this article at a reasonable length I'll forego describing the mechanics of `format_text`. Perhaps I'll write a post detailing this method in the future.

Once a book is displayed to the user `see_more_books_or_exit` and the user is given the option to start the selection over or exit the application.

Don't get hung up on trying to build something profound or find the perfect website. Pick

* Don't get hung up on trying to build something profound.
* Don't spend too much time trying to find the perfect website.
* Pick a website that approximates the tutorial examples and start coding. Pitfalls will become clear in the development process and it won't be such a big deal to start over with another site.
* Learn as much as possible about Git, GitHub, Nokogiri and Bundler.
* [Watch This](https://www.youtube.com/watch?v=XBgZLm-sdl8)
