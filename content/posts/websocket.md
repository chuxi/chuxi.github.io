+++
title = "Rust: websocket with proxy"
date = "2022-12-15"
description = """The article shows how to set up proxy on websocket stream, \
    with tokio-tungstenite library"""

[taxonomies]
tags = ["rust", "network"]

+++

## websocket

For rust websocket library [tokio-tungstenite](https://github.com/snapview/tokio-tungstenite), 
it does not support proxy of `http[s]` and `socks4/5`.
Instead, it opens the interface of accepting a proxied TCP stream. So users can quickly set up a proxy
websocket.

In the article, It looks into the proxy implementation in another rust http library 
[reqwest](https://github.com/seanmonstar/reqwest). With
the reference of design, we can easily understand how the proxy works in http.

Another important rust library is [tokio](https://github.com/tokio-rs/tokio), it offers `runtime` engine,
for executing tasks in async/await mode. It has an
[async-book](https://rust-lang.github.io/async-book/01_getting_started/02_why_async.html), 
which explains the async stream design in rust and `futures-rs` library.

The last knowledge we need to understand is proxy protocols. In addition to standard
`http[s]` and `socks4/5` protocols, it has some other similar network protocols, such as `tor`, etc.
All protocols are running on the TCP layer in network, by sending protocol-organized bytes in stream header.
The proxy server will transfer all data to the target automatically.

## InnerProxy

Let's define a proxy struct, `InnerProxy`. It parses the proxy configuration, 
and build the Tcp Connection to Proxy.
To support multiple proxy protocols, we use `enum` to distinguish each.
The `Tls` layer is already supported by `tokio-tungstenite`. After we build the Tcp stream to proxy server,
the Tcp stream will be connected by a `Tls` stream from application side. 

```rust
enum InnerProxy {
    // http or https
    Http {
        auth: Option<Vec<u8>>,
        url: String,
    },
    // socks5
    Socks {
        auth: Option<(String, String)>,
        url: String,
    }
}

impl InnerProxy {
    fn from_proxy_str(proxy_str: &str) -> Result<InnerProxy, Error> {
        use url::Position;

        let url = match Url::parse(proxy_str) {
            Ok(u) => u,
            Err(e) => return Err(Error::new(
                ErrorKind::InvalidInput, "failed to parse proxy url"))
        };
        let addr = &url[Position::BeforeHost..Position::AfterPort];

        match url.scheme() {
            "http" | "https" => {
                let mut basic_bytes: Option<Vec<u8>> = None;
                if let Some(pwd) = url.password() {
                    let encoded_str = format!("Basic {}", base64::encode(&format!("{}:{}", url.username(), pwd)));
                    basic_bytes = Some(encoded_str.into_bytes());
                };

                Ok(InnerProxy::Http {
                    auth: basic_bytes,
                    url: addr.to_string(),
                })
            },
            "socks5" => {
                let mut auth_pair = None;
                if let Some(pwd) = url.password() {
                    auth_pair = Some((url.username().to_string(), pwd.to_string()))
                };

                Ok(InnerProxy::Socks {
                    auth: auth_pair,
                    url: addr.to_string(),
                })
            }

            _ => Err(Error::new(ErrorKind::Unsupported, "unknown schema"))
        }

    }

    pub async fn connect_async(&self, target: &str) -> Result<ProxyStream, Error> {
        let target_url = Url::parse(target)
            .unwrap_or_else(|e| panic!("failed to parse target url: {}", target));
        let host = match target_url.host_str() {
            Some(host) => host.to_string(),
            None => return Err(Error::new(ErrorKind::Unsupported,
                                          "target host not available")),
        };
        let port = target_url.port().unwrap_or(443);
        match self {
            InnerProxy::Http {auth, url } => {
                let mut tcp_stream = TcpStream::connect(url).await
                    .expect("failed to connect http[s] proxy");
                Ok(ProxyStream::Http(Self::tunnel(tcp_stream, host, port, auth).await.unwrap()))
            },
            InnerProxy::Socks { auth, url} => {
                let stream = match auth {
                    Some(au) => Socks5Stream::connect_with_password(
                        url.as_str(), (host.as_str(), port), &au.0, &au.1).await,
                    None => Socks5Stream::connect(url.as_str(), (host.as_str(), port)).await,
                };
                match stream {
                    Ok(s) => Ok(ProxyStream::Socks(s)),
                    Err(e) => Err(Error::new(ErrorKind::NotConnected, "failed to create socks proxy stream"))
                }
            }
        }
    }

    async fn tunnel(mut conn: TcpStream,
                    host: String,
                    port: u16,
                    auth: &Option<Vec<u8>>) -> Result<TcpStream, Error>
    {
        use tokio::io::{AsyncReadExt, AsyncWriteExt};
        let mut buf = format!(
            "\
         CONNECT {0}:{1} HTTP/1.1\r\n\
         Host: {0}:{1}\r\n\
         ",
            host, port
        ).into_bytes();

        if let Some(au) = auth {
            buf.extend_from_slice(b"Proxy-Authorization: ");
            buf.extend_from_slice(au.as_slice());
            buf.extend_from_slice(b"\r\n");
        }

        buf.extend_from_slice(b"\r\n");
        conn.write_all(&buf).await.unwrap();

        let mut buf = [0; 1024];
        let mut pos = 0;

        loop {
            let n = conn.read(&mut buf[pos..]).await?;
            if n == 0 {
                return Err(Error::new(ErrorKind::UnexpectedEof, "0 bytes in reading tunnel"));
            }
            pos += n;

            let recvd = &buf[..pos];
            if recvd.starts_with(b"HTTP/1.1 200") || recvd.starts_with(b"HTTP/1.0 200") {
                if recvd.ends_with(b"\r\n\r\n") {
                    return Ok(conn);
                }
                if pos == buf.len() {
                    return Err(Error::new(ErrorKind::UnexpectedEof, "proxy headers too long than tunnel"));
                }
            } else if recvd.starts_with(b"HTTP/1.1 407") {
                return Err(Error::new(ErrorKind::PermissionDenied, "proxy authentication required"));
            } else {
                return Err(Error::new(ErrorKind::Other, "unsuccessful tunnel"));
            }
        }
    }
}
```

## `ProxyStream`

To support multiple protocols, we also need to return a stream object.
Here we can have two kinds of design with the stream object. First, we can use `Polymorphism` in Rust,
`type BoxConn = Box<dyn SomeConn>` and `trait SomeConn: AsyncRead + AsyncWrite + Unpin {}`. 
The `reqwest` library used the solution in its proxy implementation. The second, using `enum` and
to each protocol stream, implementing `AsyncRead` and `AsyncWrite` traits. Apparently, the second is better.
because dynamical dispatching will cut down the performance. 


```rust
enum ProxyStream {
    Http(TcpStream),

    Socks(Socks5Stream<TcpStream>)
}

impl AsyncRead for ProxyStream {
    fn poll_read(self: Pin<&mut Self>,
                 cx: &mut Context<'_>,
                 buf: &mut ReadBuf<'_>) -> Poll<std::io::Result<()>> {
        match self.get_mut() {
            ProxyStream::Http(s) => Pin::new(s).poll_read(cx, buf),
            ProxyStream::Socks(s) => Pin::new(s).poll_read(cx, buf),
        }
    }
}

impl AsyncWrite for ProxyStream {
    fn poll_write(self: Pin<&mut Self>,
                  cx: &mut Context<'_>,
                  buf: &[u8]) -> Poll<Result<usize, Error>> {
        match self.get_mut() {
            ProxyStream::Http(s) => Pin::new(s).poll_write(cx, buf),
            ProxyStream::Socks(s) => Pin::new(s).poll_write(cx, buf),
        }
    }

    fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Error>> {
        match self.get_mut() {
            ProxyStream::Http(s) => Pin::new(s).poll_flush(cx),
            ProxyStream::Socks(s) => Pin::new(s).poll_flush(cx),
        }
    }

    fn poll_shutdown(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Error>> {
        match self.get_mut() {
            ProxyStream::Http(s) => Pin::new(s).poll_shutdown(cx),
            ProxyStream::Socks(s) => Pin::new(s).poll_shutdown(cx),
        }
    }
}
```

## Example

In `cargo.toml`, add the dependencies

```toml
tokio = { version = "1", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0.89"
tokio-tungstenite = { version = "0.18", features = [ "native-tls" ]}
futures-util = "0.3"
tokio-socks = "0.5.1"
url = "2.3.1"
base64 = "0.20.0"
```

In `websocket_test.rs`

```rust
use std::borrow::Borrow;
use std::fmt::format;
use std::io::Error;
use std::io::ErrorKind;
use std::net::SocketAddr;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Instant, SystemTime};

use futures_util::sink::SinkExt;
use futures_util::StreamExt;
use serde::Serialize;
use tokio::io::{AsyncRead, AsyncWrite, AsyncWriteExt, ReadBuf};
use tokio::net::TcpStream;
use tokio_socks::TargetAddr;
use tokio_socks::tcp::Socks5Stream;
use tokio_tungstenite::client_async_tls;
use tokio_tungstenite::Connector::NativeTls;
use tokio_tungstenite::tungstenite::{connect, HandshakeError, Message};
use url::Url;

use exchange_api::add;

fn main() {
    // websocket requires proxy
    let websocket_addr = "wss://stream.binance.com/ws";

    let runtime = tokio::runtime::Builder::new_multi_thread()
        .enable_all().build().expect("failed to create runtime");

    runtime.block_on(async move {
        let proxy_url = "socks5://127.0.0.1:1080";
        // let proxy_url = "http://127.0.0.1:1081";
        let proxy = InnerProxy::from_proxy_str(proxy_url)
            .expect("failed to parse inner proxy");

        let mut tcp_stream = proxy.connect_async(websocket_addr).await
            .unwrap_or_else(|e| panic!("failed to create proxy stream: {}", e));

        let (mut stream, resp) = match client_async_tls(
            websocket_addr, tcp_stream).await {
            Ok(res) => res,
            Err(e) => {
                match e {
                    tokio_tungstenite::tungstenite::Error::Http(re) => {
                        println!("{}", String::from_utf8_lossy(re.into_body().unwrap().as_slice()));
                    }
                    _ => println!("{:?}", e),
                }
                return;
            }
        };

        let sub = Subscribe::new(
            vec!["btcusdt@aggTrade".to_string()],
            SystemTime::now().duration_since(SystemTime::UNIX_EPOCH).unwrap().as_secs());
        let sub_string = serde_json::to_string(&sub).unwrap();
        println!("sub_string: {}", sub_string);
        stream.send(Message::Text(sub_string)).await;

        while let Some(msg) = stream.next().await {
            match msg {
                Ok(_msg) => println!("message: {:?}", _msg),
                Err(e) => println!("message: {:?}", e)
            }
        }
    });

}


#[derive(Debug, Serialize)]
struct Subscribe {
    method: String,
    params: Vec<String>,
    id: u64,
}

impl Subscribe {
    pub fn new(params: Vec<String>, id: u64) -> Self {
        Self { method: "SUBSCRIBE".to_string(),
            params, id }
    }
}


```

## Summary

The official `tokio-tungstenite` library has some issues about the proxy feature support, such as 
[issue - Is there any support for proxies? #177](https://github.com/snapview/tungstenite-rs/issues/177).

So with this article, we can add proxy feature to `tokio-tungstenite` library. Wish it helps you.

