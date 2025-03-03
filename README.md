# go-pop3

A simple Go POP3 client library for connecting and reading mails from POP3 servers. This is a full rewrite of [TheCreeper/go-pop3](https://github.com/TheCreeper/go-pop3) with bug fixes and new features.


## Install
`go get -u github.com/snekROmonoro/go-pop3`


## Example
```go
package main

import (
	"fmt"
	"github.com/snekROmonoro/go-pop3"
)

func main() {
	// Initialize the client.
	p := pop3.New(pop3.Opt{
		Host: "pop.gmail.com",
		Port: 995,
		TLSEnabled: true,
	})

	// Create a new connection. POP3 connections are stateful and should end
	// with a Quit() once the opreations are done.
	c, err := p.NewConn()
	if err != nil {
		log.Fatal(err)
	}
	defer c.Quit()

	// Authenticate.
	if err := c.Auth("myuser", "mypassword"); err != nil {
		log.Fatal(err)
	}

	// Print the total number of messages and their size.
	count, size, _ := c.Stat()
	fmt.Println("total messages=", count, "size=", size)

	// Pull the list of all message IDs and their sizes.
	msgs, _ := c.List(0)
	for _, m := range msgs {
		fmt.Println("id=", m.ID, "size=", m.Size)
	}

	// Pull all messages on the server. Message IDs go from 1 to N.
	for id := 1; id <= count; id++ {
		m, _ := c.Retr(id)
		mr := mail.NewReader(m)

		subject, err := mr.Header.Subject()
		if err != nil {
			fmt.Printf("failed to read message subject: %v\n", err)
			continue
		}

		fmt.Println(id, "=", subject)

        for {
			p, err := mr.NextPart()
			if err == io.EOF {
				break
			} else if err != nil {
				log.Printf("failed to read message part: %v\n", err)
				continue
			}
            
			switch p.Header.(type) {
			case *mail.InlineHeader:
				b, err := io.ReadAll(p.Body)
				if err != nil {
					log.Printf("failed to read message body: %v\n", err)
				} else {
					log.Printf("body: %s\n", string(b))
				}
			}
		}
	}

	// Delete all the messages. Server only executes deletions after a successful Quit()
	for id := 1; id <= count; id++ {
		c.Dele(id)
	}
}
```
