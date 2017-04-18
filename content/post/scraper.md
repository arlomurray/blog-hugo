+++
date = "2017-04-16T12:53:14+09:00"
title = "Web Scraping Go Docs for Error Messages"
tags = [
      "golang"      
]
+++
## Background
Earlier in the year, there was quite a kerfuffle surrounding the data archived on government sites. With the new administration, scientists and concerned citizens were worried about what would happen to the information on government websites and started to salvage as much of it as they could. Events started being organized, primarily at universities, where people would volunteer to scour these sites for important data. This process is known as *web scraping*.

The idea of making a web scraper has since stayed in my head. The problem was, I didn't know what I wanted it to be able to do. There are obviously easy programs one could build, but I wanted to make one that was at least semi-cool. Yesterday, when I was thinking back on some especially tricky error messages I've come across, I thought of a worthy enough idea. Essentially, the web scraper would scour the Go standard library files for the error message I had encountered. Yes, Google does the same thing, but this (in my mind) is a much cooler way to go about it.

## What the Web Scraper will do
The process the program will execute will be as follows:

1. Take in the error message as an argument
2. Get the Go Docs homepage
3. Scrape the documentation for source file links
4. For each link, search the file associated with the link for the error message
5. Return the line number of the error message

Go documentation, as you probably know, is incredibly well organized and formatted. The standard library packages in particular are very easy to navigate and search through. Links to each of the package files can be found at https://golang.org/src.

<center><img src="/images/scraper1.png" style="width:100%;"></center>

All the source files of each package are also listed under https://golang.org/src/<PACKAGENAME>/.

<center><img src="/images/scraper2.png" style="width:100%;"></center>

The docs are also organized in a way that when a function's link is clicked, the user will be lead to the line number the function appears in the source file. That url looks like this: https://golang.org/src/fmt/print.go#L256, where "#L" specifies the line number. However, files from non-standard library packages are not stored on golang.org. This applies to Go's "x" packages as well. Due to this, the web scraper will only be capable of searching through standard lib files.

<center><img src="/images/scraper3.png" style="width:100%;"></center>

The scraper is also going to assume that the user knows which package the error message came from. Although, it wouldn't require much more code to implement that functionality. The package name, along with the error message, will be taken in as a command line argument.

## Opening the Web Page
Using the `Get` function from the "net/http" package, we'll make a call to golang.org/src/<PACKAGE NAME>.
```
resp, err := http.Get("https://golang.org/src/" + os.Args[1])
if err != nil {
  log.Fatal("could not get package files")
}
defer resp.Body.Close()
```

## Tokenizer
To do the actual scraping, we can use one of Go's "x" packages called [net/html](https://godoc.org/golang.org/x/net/html). With this package, our program will be able to parse the HTML on a certain web page and search for all of the `<a>` HTML tags.The `NewTokenizer` method will do the searching. It creates a *tokenizer* for the HTML, which categorizes the HTML into token types. Tokens are HTML tags. These tokens can then in turn, have their attributes evaluated.

*Types of Tokens HTML tags can be evaluated as*
```
const (
  // ErrorToken means that an error occurred during tokenization.
  ErrorToken TokenType = iota
  // TextToken means a text node.
  TextToken
  // A StartTagToken looks like <a>.
  StartTagToken
  // An EndTagToken looks like </a>.
  EndTagToken
  // A SelfClosingTagToken tag looks like <br/>.
  SelfClosingTagToken
  // A CommentToken looks like <!--x-->.
  CommentToken
  // A DoctypeToken looks like <!DOCTYPE x>
  DoctypeToken
)
```
Since the program will be searching for `<a>` tags, it will technically be looking for `StartTagToken`s.  The program could search through a web page in a for loop, however, if the page for some reason does not have a link, we would rather have the the program end instead of running infinitely. Coming across an `ErrorToken` would be a good indication of the program hitting the end of the web page.
```
tizer := html.NewTokenizer(resp.Body)
var urls []string
loop:
for {
  token := tizer.Next()
  switch {
  case token == html.ErrorToken:
    break loop
  case token == html.StartTagToken:
    found := tizer.Token()
    if found.Data != "a" {
      continue
    }
    for _, a := range found.Attr {
      if a.Key == "href" {
        urls = append(urls, a.Val)
      }
    }
  }
}
```
`NewTokenizer` takes in a `io.Reader` as it's parameter, so we are able to pass in the HTTP response body. `NewTokenizer` only splits the HTMl into tokens; in order to iterate over each token, the `Next()` method from "net/html" package is used. Each token has [fields](https://godoc.org/golang.org/x/net/html#Token) that can be referenced, and from the fields we can extract the link to each file in a particular package. In the program, we use a for loop to iterate over the token's fields until we come across the `href` field. Once we have the value of `href`, we append it to the slice of URLs that was initialized outside of the for loop and that slice can be sent to another process which iterates over each one in an attempt to find the error message.

## Finding the Error Message
To read the contents of each file, the body should be scanned line by line. We can do that by using the scanner from package "bufio". While scanning the body, we can call the `Contains()` method from package "strings" to search for the error message. If the program comes across it, the line will be tokenized. This time, we are searching for the line number to return to the user. If you look at the file's HTML, you will see that each line number is placed in a set of `<span>` tags and is also set as the `id` attribute. Once we have that, the program can open a browser window to the URL specified with the line number.

Note: Having the scanner finish scanning the whole body, instead of stopping when it comes across the first instance of an error message is pretty helpful if whatever the user is searching for shows up more than once in the file.
```
for _ ,url := range urls {
  resp, err := http.Get("https://golang.org/src/" + os.Args[1] + "/" + url)
  if err != nil {
    fmt.Println("could not get package file")
  }
  defer resp.Body.Close()

  scanner := bufio.NewScanner(resp.Body)
  for scanner.Scan() {
    line := scanner.Text()
    if strings.Contains(line, os.Args[2]) {
      tizer := html.NewTokenizer(strings.NewReader(line))
      token := tizer.Next()

      switch {
      case token == html.ErrorToken:
        break
      case token == html.StartTagToken:
        found := tizer.Token()
        if found.Data != "span" {
          continue
        }

        for _, a := range found.Attr {
          if a.Key == "id" {
            cmd := exec.Command("open", (resp.Request.URL).String() + "#" + a.Val)
            cmd.Run()
          }
        }
      }
    }
  }
}
```
## Separating the Program Into Functions
Aside from `func main`, the program could be split into two other functions. One function could handle scraping the `<a>` tags from the package's source page, while the other could handle searching for the error messages from the file itself.
```
package main

import (
  "golang.org/x/net/html"
  "net/http"
  "os"
  "log"
  "os/exec"
  "bufio"
  "strings"
)

func main() {
    urls := crawl()
    crawlEach(urls)
}
func crawl() []string {
  resp, err := http.Get("https://golang.org/src/" + os.Args[1])
  if err != nil {
    log.Fatal("could not get package files")
  }

  defer resp.Body.Close()
  tizer := html.NewTokenizer(resp.Body)
  var urls []string
  loop:
  for {
    token := tizer.Next()
    switch {
    case token == html.ErrorToken:
      break loop
    case token == html.StartTagToken:
      found := tizer.Token()
      if found.Data != "a" {
        continue
      }
      for _, a := range found.Attr {
        if a.Key == "href" {
          urls = append(urls, a.Val)
        }
      }
    }
  }
}
func crawlEach(urls []string) {
  for _ ,url := range urls {
    resp, err := http.Get("https://golang.org/src/" + os.Args[1] + "/" + url)
    if err != nil {
      log.Fatal("could not get package file")
    }
    defer resp.Body.Close()

    scanner := bufio.NewScanner(resp.Body)
    for scanner.Scan() {
      line := scanner.Text()
      if strings.Contains(line, os.Args[2]) {
        tizer := html.NewTokenizer(strings.NewReader(line))
        token := tizer.Next()

        switch {
        case token == html.ErrorToken:
          break
        case token == html.StartTagToken:
          found := tizer.Token()
          if found.Data != "span" {
            continue
          }

          for _, a := range found.Attr {
            if a.Key == "id" {
              cmd := exec.Command("open", (resp.Request.URL).String() + "#" + a.Val)
              cmd.Run()
            }
          }
        }
      }
    }
  }
}
```
There are quite a few bugs with this code, on top of being unorganized. For instance, if a user were to search for "if" in the source files, the program would pick up possibly hundreds of results, and if it were to try and open a new browser window for every instance, the result would be total mayhem. Overall though, it's a pretty cool way to search for error messages. Let me know what you think in the comments, or shoot me an [email](mailto:arloemurray@gmail.com?Subject=Web%20scraper). The code can also be viewed on my [Github](https://github.com/arlomurray/godoc-scraper) profile.
