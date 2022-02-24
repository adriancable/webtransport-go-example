# webtransport-go-example

An example WebTransport-over-HTTP/3 server in Go, and browser client in JavaScript, using [`github.com/adriancable/webtransport-go`](https://github.com/adriancable/webtransport-go).

The server app demonstrates how to create a WebTransport server, accept an incoming session, accept client-initiated and create server-initiated bidirectional and unidirectional streams, and send and receive data.

The browser client app demonstrates how to create a WebTransport connection to the server, create some streams, send data back and forth, and then close the transport.

## Get a certificate

You'll need to get a certificate for your server. Please read the comments at the top of [Google's WebTransport server example in Python](https://github.com/GoogleChrome/samples/blob/gh-pages/webtransport/webtransport_server.py) for detailed instructions. Then, put your `cert.pem` and `cert.key` files in the same directory that you've checked this example out in.

## Run the server

`go run main.go`

You should see:

`Launching WebTransport server at :4433`

## Run the client

Easiest: launch Chrome, then File / Open File and select `client.html`.

More representative: spin up an HTTP server on port 8000, then point Chrome at `http://localhost:8000/client.html`. If you change the host or the port
you're serving from, you'll need to adjust `AllowedOrigins` in `main.go` accordingly.

## The server should display ...

```
Accepted incoming WebTransport session
Listening on server-initiated bidi stream 1
Sending to server-initiated bidi stream 1: bidi
Sending to server-initiated uni stream 1: uni
Accepting incoming bidi stream: 4
Accepting incoming uni stream: 6
Accepting incoming uni stream: 10
Accepting incoming uni stream: 14
Received datagram: Datagram
Sending datagram: DATAGRAM
Received from server-initiated bidi stream 1: BIDI
Received from uni stream: Uni stream
Received from bidi stream 4: Bidi stream
Sending to bidi stream 4: BIDI STREAM
Received datagram: Datagram again
Sending datagram: DATAGRAM AGAIN
Received from bidi stream 4: Bidi stream again
Received from uni stream: Uni stream again
Sending to bidi stream 4: BIDI STREAM AGAIN
2022/02/23 18:03:28 Error reading from bidi stream 4: EOF
Session closed, ending datagram listener: WebTransport stream closed
Session closed, not accepting more bidi streams: context canceled
Session closed, not accepting more uni streams: context canceled
2022/02/23 18:03:30 Error reading from uni stream 14: stream 14 canceled with error code 271
2022/02/23 18:03:30 Error reading from server-initiated bidi stream 1: stream 1 canceled with error code 271
```

## The browser should display in the console ...

```
Received an incoming bidi stream
client.html:61 Received an incoming uni stream
client.html:56 Received on incoming bidi stream: bidi
client.html:63 Received on incoming uni stream: uni
client.html:46 Received datagram: DATAGRAM
client.html:49 Received on bidi stream: BIDI STREAM
client.html:46 Received datagram: DATAGRAM AGAIN
client.html:49 Received on bidi stream: BIDI STREAM AGAIN
client.html:17 Datagram receive error: WebTransportError: The session is closed.
    at main (client.html:82:21)
client.html:17 Incoming bidi stream receive error: WebTransportError: The session is closed.
    at main (client.html:82:21)
client.html:17 Incoming uni stream receive error: WebTransportError: The session is closed.
    at main (client.html:82:21)
client.html:30 Connection closed normally
```

Take a look at the [webtransport-go-example](https://github.com/adriancable/webtransport-go-example) repo for a complete server (and browser client) example.

## Minimal "getting started" example

You'll need to get a certificate for your server. Please read the comments at the top of [Google's WebTransport server example in Python](https://github.com/GoogleChrome/samples/blob/gh-pages/webtransport/webtransport_server.py) for detailed instructions.

First, set up WebTransport server parameters:

```
server := &webtransport.Server{
	ListenAddr:     ":4433",
	TLSCert:        webtransport.CertFile{Path: "cert.pem"},
	TLSKey:         webtransport.CertFile{Path: "cert.key"},
	AllowedOrigins: []string{"googlechrome.github.io", "127.0.0.1:8000", "localhost:8000"},
}
```

Then, set up an `http.Handler` to accept a session, wait for an incoming bidirectional stream from the client, then (in this example) receive data and echo it back:

```
http.HandleFunc("/counter", func(rw http.ResponseWriter, r *http.Request) {
	session := r.Body.(*webtransport.Session)
	session.AcceptSession()
	defer session.CloseSession()

	// Wait for incoming bidi stream
	s, err := session.AcceptStream()
	if err != nil {
		return
	}

	for {
		buf := make([]byte, 1024)
		n, err := s.Read(buf)
		if err != nil {
			break
		}
		fmt.Printf("Received from bidi stream %v: %s\n", s.StreamID(), buf[:n])
		sendMsg := bytes.ToUpper(buf[:n])
		fmt.Printf("Sending to bidi stream %v: %s\n", s.StreamID(), sendMsg)
		s.Write(sendMsg)
	}
}
```

Finally, start the server:

```
ctx, cancel := context.WithCancel(context.Background())
server.Run(ctx)
```

Here is a simple Chrome browser client to talk to this server. You can open a new browser tab and paste it into the Chrome DevTools console:

```
let transport = new WebTransport("https://localhost:4433/counter");
await transport.ready;
let stream = await transport.createBidirectionalStream();
let encoder = new TextEncoder();
let decoder = new TextDecoder();
let writer = stream.writable.getWriter();
let reader = stream.readable.getReader();
await writer.write(encoder.encode("Hello, world!"))
console.log(decoder.decode((await reader.read()).value));
transport.close();
```

## License
Provided under the MIT license.
