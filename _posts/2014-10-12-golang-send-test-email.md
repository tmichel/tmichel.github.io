---
layout: post
title: Sending and testing emails in Go
date: 2014-10-12 14:00:00 CEST
---

My last project was to create a notification service for our back-end. It had to
support a few different ways to send out notifications and of course sending
email was among them. Go's awesome standard library makes it super easy to send
out emails. After some initialization it comes down to one function call. For
example sending email via Gmail would look something like this:

~~~go
from := "you@gmail.com"
auth := smtp.PlainAuth("", from, "<your gmail password>", "smtp.gmail.com")
err := smtp.SendMail(
    "smtp.gmail.com:587",               // server address
    auth,                               // authentication
    from,                               // sender's address
    []string{"recipient@example.com"},  // recipients' address
    []byte("Hello World!"),             // message body
)
if err != nil {
    log.Print(err)
}
~~~

This is all good but what about testing? It is good practice _not to test_ your
framework and standard library again, but what about all the other stuff?
Building the email's body, loading configuration and all the necessary
preparation - that has to be tested somehow.

You could use an existing SMTP server (such as Gmail) but introducing external
dependencies to your tests are a bad idea. You might think that mocking the SMPT
server by spinning up a local from your test case could solve your problem. It
certainly would but it is unnecessarily difficult and time consuming. Go has two
useful features that really helps with testing. Interfaces and first-class
functions.

Let's create an interface for our email sender!

~~~go
type EmailSender interface {
    Send(to []string, body []byte) error
}
~~~

This way we can mock the entire email sending later when we use it as a
dependency and this approach also provide us a convenient way to test emailing
and well.

Let's create a simple implementation for this interface!

~~~go
type EmailConfig struct {...}

type emailSender struct {
    conf EmailConfig
    send func(string, smtp.Auth, string, []string, []byte) error
}

func (e *emailSender) Send(to []string, body []byte) error {
    addr := e.conf.ServerHost + ":" + e.conf.ServerPort
    auth := smtp.PlainAuth("", e.conf.Username, e.conf.Password, e.conf.ServerHost)
    return e.send(addr, auth, e.conf.SenderAddr, to, body)
}
~~~

There is no call to `smtp.SendMail` yet, although in a constructor function we
can easily wire it together. We pass the a function reference to `smpt.SendMail`
to the a newly created `emailSender` instance - the outside world will never
know the difference.

~~~go
func NewEmailSender(conf EmailConfig) EmailSender {
    return &emailSender{conf, smtp.SendMail}
}
~~~

Now we can easily mock out the actual email sending function in our tests.
Creating something similar to `httptest.ResponseRecorder` we can actually see
what was passed to the sender function.

~~~go
func mockSend(errToReturn error) (func(string, smtp.Auth, string, []string, []byte) error, *emailRecorder) {
    r := new(emailRecorder)
    return func(addr string, a smtp.Auth, from string, to []string, msg []byte) error {
        *r = emailRecorder{addr, a, from, to, msg}
        return errToReturn
    }, r
}

type emailRecorder struct {
    addr string
    auth smtp.Auth
    from string
    to   []string
    msg  []byte
}
~~~

Using a handy closure here we can record what was sent and we can emulate an
error for the edge cases.

Now all we have to do is put everything together in a test:

~~~go
func TestEmail_SendSuccessful(t *testing.T) {
    f, r := mockSend(nil)
    sender := &emailSender{send: f}
    body := "Hello World"
    err := sender.Send([]string{"me@example.com"}, []byte(body))

    if err != nil {
        t.Errorf("unexpected error: %s", err)
    }
    if string(r.msg) != body {
        t.Errorf("wrong message body.\n\nexpected: %\n got: %s", body, r.msg)
    }
}
~~~

[All to code stiched together](https://gist.github.com/tmichel/a57bd4033db3e90f5516).
Download it and run `go test email_test.go` to run the test.
