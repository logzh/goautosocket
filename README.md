# GoAutoSocket (GAS) ![Status](https://img.shields.io/badge/status-stable-green.svg?style=plastic) [![Build Status](http://img.shields.io/travis/teh-cmc/goautosocket.svg?style=plastic)](https://travis-ci.org/teh-cmc/goautosocket) [![GoDoc](http://img.shields.io/badge/go-documentation-blue.svg?style=plastic)](http://godoc.org/github.com/teh-cmc/goautosocket)

The GAS library provides auto-reconnecting TCP sockets in a tiny, fully tested, thread-safe API.

The `TCPClient` struct embeds a `net.TCPConn` and overrides its `Read()` and `Write()` methods, making it entirely compatible with the `net.Conn` interface and the rest of the `net` package.

## Install

```bash
get -u github.com/teh-cmc/goautosocket
```

## Usage

To test the library, you can run a local TCP server with:

    $ tcpserver -v -RHl0 127.0.0.1 9999 echo

and run this code:

```go
package main

import (
    "log"
    "time"

    "github.com/teh-cmc/goautosocket"
)

func main() {
    // connect to a TCP server
    conn, err := gas.Dial("tcp", "localhost:9999")
    if err != nil {
        log.Fatal(err)
    }

    // client sends "hello, world!" to the server every second
    for {
        _, err := conn.Write([]byte("hello, world!"))
        if err != nil {
            // if the client reached its retry limit, give up
            if err == gas.ErrMaxRetries {
                log.Println("client gave up, reached retry limit")
                return
            }
            // not a GAS error, just panic
            log.Fatal(err)
        }
        log.Println("client says hello!")
        time.Sleep(time.Second)
    }
}
```

Then try to kill and reboot your server, the client will automatically reconnect and start sending messages again; unless it has reached its retry limit.

## Examples

An advanced example of a client writing to a buggy server that's randomly crashing and rebooting:

```go
package main

import (
    "log"
    "math/rand"
    "net"
    "sync"
    "time"

    "github.com/teh-cmc/goautosocket"
)

func main() {
    // open a server socket
    s, err := net.Listen("tcp", "localhost:0")
    if err != nil {
        log.Fatal(err)
    }
    // save the original port
    addr := s.Addr()

    // connect a client to the server
    c, err := gas.Dial("tcp", s.Addr().String())
    if err != nil {
        log.Fatal(err)
    }
    defer c.Close()

    // shut down and boot up the server randomly
    var swg sync.WaitGroup
    swg.Add(1)
    go func() {
        defer swg.Done()
        for i := 0; i < 5; i++ {
            log.Println("server up")
            time.Sleep(time.Millisecond * 100 * time.Duration(rand.Intn(20)))
            if err := s.Close(); err != nil {
                log.Fatal(err)
            }
            log.Println("server down")
            time.Sleep(time.Millisecond * 100 * time.Duration(rand.Intn(20)))
            s, err = net.Listen("tcp", addr.String())
            if err != nil {
                log.Fatal(err)
            }
        }
    }()

    // client writes to the server and reconnects when it has to
    // this is the interesting part
    var cwg sync.WaitGroup
    cwg.Add(1)
    go func() {
        defer cwg.Done()
        for {
            if _, err := c.Write([]byte("hello, world!")); err != nil {
                switch e := err.(type) {
                case gas.Error:
                    if e == gas.ErrMaxRetries {
                        log.Println("client leaving, reached retry limit")
                        return
                    }
                default:
                    log.Fatal(err)
                }
            }
            log.Println("client says hello!")
        }
    }()

    // terminates the server indefinitely
    swg.Wait()
    if err := s.Close(); err != nil {
        log.Fatal(err)
    }

    // wait for the client to give up
    cwg.Wait()
}
```

You can also find an example with concurrency [here](https://github.com/teh-cmc/goautosocket/blob/master/tcp_client_test.go#L97).

## Disclaimer

This was built with my needs in mind, no more, no less. That is, I needed a simple and thread-safe API to talk concurrently, through a TCP socket, to some services of mine which are might be rebooted at any time (and oftentimes against my will).

Surprisingly, I couldn't find such a library (I guess I either didn't look in the right place, or just not hard enough.. oh well); so here it is.
Do not hesitate to send a pull request if this doesn't cover all your needs (and it probably won't), there are more than welcome.

## License ![License](https://img.shields.io/badge/license-MIT-blue.svg?style=plastic)

The MIT License (MIT) - see LICENSE for more details

Copyright (c) 2015  Clement 'cmc' Rey  <cr.rey.clement@gmail.com>
