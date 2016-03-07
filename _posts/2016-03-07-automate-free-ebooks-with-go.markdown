---
layout: post
title:  "Automate Your Free eBook Download with Go"
date:   2016-03-07 14:10:22 -0500
categories: go programming
---
[Packt Publishing][packt] is one of many online book vendors geared toward programmers, but distinguishes itself by offering Packt account holders (free to sign up) one free ebook per day. Packt's [Free Learning website][packt-free] gives your 24 hours to snag the free book-of-the-day; it even has a helpful counter to let you know how long you have to grab it (or how long you have to wait for the next book to appear). The books are selected randomly, and their topics vary widely. Today's free book is about SQL Server 2012, while yesterday's was AngularJS Directives. Interestingly, Packt will occasionally have themed, week-long book giveaways, such as Game Development week or AngularJS week.

I've been grabbing free books on and off for the past year, mostly when I simultaneously remembered to check and found something interesting. But remembering is an error-prone task, so when I started learning Go, and took it upon myself to automate the process.

The end goal was to create a small Go CLI tool that would run once a day. This script would check the [free learning website][packt-free], grab the free book-of-the-day's title, and determine whether or not I was interested in it. Lastly, and most importantly, if I was interested in the book, the tool would send me a notification.

Overall, this was a straightforward process. That said, there were a number of decisions I had to make to get the application to work, including:

* How would I judge whether or not I was interested in the book-of-the-day?
* How would would I notify myself of the book's title?
* How would the tool be packaged and run?

I decided to answer each of these questions as simply as possible. This made the project easier to implement, but did have some down sides, which I will share at the end of this post.

## Implementation

First off, I grabbed the book's title by grabbing HTML from the Free Learning page and parsing it for the first header (which is always the book's title). In this process, I learned about tokenizers, which essentially split an HTML document into "tokens", which can then be iterated over. I used the `golang.org/x/net/html` package to implement this. If this sounds of particular interest to you (perhaps you want to do web scraping with Go?), you can read more about [the html package][go-html-package]. 

So first, I iterated over the HTML to grab all of the headers. 

{% highlight go %}
tokenizer := html.NewTokenizer(body)
counter := 0
for {
    tokenType := tokenizer.Next()
    token := tokenizer.Token()

    switch tokenType {
    case html.ErrorToken:
        return headers
    case html.StartTagToken:
        if token.Data == "h2" {
            counter++
        }
    case html.TextToken:
        if counter > 0 {
            counter--
            headers = append(headers, token.Data)
        }
    case html.EndTagToken:
        continue
}
{% endhighlight %}

Then grabbed the first element from the headers slice, and split it into strings. I used the `strings` library for this.
{% highlight go %}
bookTitleHeader := headers[0]
titleWords := strings.Fields(bookTitleHeader)
{% endhighlight %}

With the book title words in hand, the first design decision I had to make was how to determine whether or not I wanted to download the book. For simplicity's sake,I decided to compare the book's title words to a list of words that correspond to topics I'm interested in (e.g. "Python"). I stored this in a slice of strings:

{% highlight go %}
keywords := []string{"DevOps", "Go", "Golang", "Python",
        "JavaScript", "AngularJS", "Ember.js", "Cloud", "Continuous Integration", "Puppet"}
{% endhighlight %}

I then compared each word in the book's title, and if there was a word match, I would decide to download the book: 
{% highlight go %}
for download == false {
    for _, i := range bookWords {
        for _, k := range desiredKeywords {
             len := len(i)
             if i[len-1:len] == ":" && len > 2 {
                 i = i[:len-1]
             }
             if i == k {
                 download = true
                 break
             }
         }
     }
}
{% endhighlight %}

I implemented some hackery here, because `strings.Fields()` the title of the book of the day had a colon in it, so I removed it before making comparisons. In a more robust scenario, I would want to check for and remove other punctuation as well (e.g. ",", "!", etc.). I'd probably want to break this out into it's own function.

Also worth noting is that I added a "download" flag to the function, such that the comparison would be skipped, and the response would always return to download the book.

Assuming there would be cases where I did, in fact, want to download the book, I needed a way to notify myself of the book. Ultimately, I settled on sending a notification via email. Upon making this decision, I saw that Gmail offers a [free SMTP server][gmail-smtp] that lets you senf email to anyone from anywhere using your Gmail email address. Implementing this was a little bit annoying, but luckily, someone else had already done something similar, and posted a gist on [their Github][go-smtp-example]. To use `smtp.gmail.com`, you must use SSL, which made the Go implementation more interesting:

To work correctly, I needed to first create a TLS config:
{% highlight go %}
tlsconfig := &tls.Config{
    InsecureSkipVerify: true,
    ServerName:         host,
}
{% endhighlight %}

Then, I needed to create a TLS connection:
{% highlight go %}
conn, err := tls.Dial("tcp", fullAddr, tlsconfig)
checkError(err)
{% endhighlight %}

Using that connection, I then needed to create a new STMP client (using that TLS connection)...
{% highlight go %}
c, err := smtp.NewClient(conn, host)
checkError(err)
{% endhighlight %}

Give the client the needed information...
{% highlight go %}
if err = c.Auth(auth); err != nil {
    log.Panic(err)
}

if err = c.Mail(email.From.Address); err != nil {
    log.Panic(err)
}

if err = c.Rcpt(email.To.Address); err != nil {
    log.Panic(err)
}
{% endhighlight %}

And finally, write the data and close the connection. 
{% highlight go %}
    w, err := c.Data()
    checkError(err)

    _, err = w.Write([]byte(message))
    checkError(err)

    err = w.Close()
    checkError(err)
{% endhighlight %}

Another implementation detail I chose was to pass in the username and password for the Gmail account as command line flags. This was an attempt to keep configuration metadata and secrets out of source control, and was implemented using the `flags` library:
{% highlight go%}
flag.StringVar(&destinationEmail, "email", "", "for gmail")
flag.StringVar(&password, "password", "", "for gmail")
flag.Parse()
{% endhighlight %}

Lastly, an external dependency I had to configure was my Gmail account. In order for the STMP server to send mail from my Gmail, I had to turn "Allow less secure apps" ON in my privacy settings. To do this:
1. Click on the Gmail avatar picture in the upper-right corner of Gmail
2. Click the blue "My Account" button
3. Click the ">" to the right of "Sign-in and Security"
4. Click "Device activity & notifications" under "Sign-in & Security" on the left-hand toolbar
5. Scroll down to "Allow less secure apps"
6. Slide the bar to the right to "ON"

Which comes to the last decision: how to run the code itself. Fortunately, Go makes code incredibly easy to package and run, as it can be easily built into a binary. So, how to run the binary?
I chose to use a simple cron job. It runs at 11am every day (one of the times I'll likely be online on weekends as well as weekdays):
To set up the cron job, type `crontab -e`. In the editor that pops up, enter in the job:
`0 11 * * * /path/to/binary -email example@fillmeout.com -password This1s4T3STPasswd`

## Pitfalls of the Current Implementation

There are a number of things I would improve if this script were to become more widely used. They include:

* A more secure way to pass in password information than a plain-text command line flag.
* A way to change the keywords without needing to rebuild the package. Perhaps a command line flag.
* A more robust mechanism for determining interest in the book. Perhaps also grabbing the information from the book description.
* More robust handling of punctuation in titles
* More performant implementation of determining whether or not to download. Currently, the worst-case scenario is an O(n^2) implementation.

That said, this was the first "project" I attempted to complete using Go. You can find the entire source file on my [Github][github]. 

[packt]: https://packtpub.com
[packt-free]: https://packtpub.com/packt/offers/free-learning
[github]: https://gist.github.com/tyrostone/d7a2e03f091cb50eb2ac
[go-html-package]: https://godoc.org/golang.org/x/net/html
[go-smtp-example]: https://gist.github.com/chrisgillis/10888032
[gmail-smtp]: https://support.google.com/a/answer/176600?hl=en
