# ztls

[![Zeta](https://img.shields.io/badge/Zeta-Language-%23FF6B6B?style=flat-square)](https://zeta-lang.org)
[![zorbs.io](https://img.shields.io/badge/zorbs.io-@crypto/ztls-%2300C4FF?style=flat-square)](https://zorbs.io/@crypto/ztls)
[![GitHub](https://img.shields.io/badge/GitHub-murphsicles/ztls-%23181717?style=flat-square)](https://github.com/murphsicles/ztls)

Pure Zeta TLS 1.2 and 1.3 implementation. ZTLS provides encrypted communication channels — enabling HTTPS servers, secure client connections, and authenticated peer-to-peer networking without any external C library dependencies.

## Quick Start

Add to your `zorb.toml`:

```toml
[dependencies]
@crypto/ztls = "1.0"
```

### TLS Client (connect to an HTTPS server)

```zeta
use @crypto/ztls::{ClientConfig, ServerName, ClientConnection, Stream};
use @crypto/ztls::crypto::ring::default_provider;
use std::sync::Arc;
use std::net::TcpStream;

fn main() {
    // Set up TLS client config with default crypto provider
    let config = ClientConfig::builder_with_provider(default_provider().into())
        .with_safe_defaults()
        .with_root_certificates(root_store)
        .with_no_client_auth();

    // Connect TCP
    let tcp = TcpStream::connect("example.com:443").unwrap();
    let server_name = ServerName::try_from("example.com").unwrap();

    // Wrap in TLS
    let mut client = ClientConnection::new(Arc::new(config), server_name).unwrap();
    let mut tls = Stream::new(&mut client, tcp);
    tls.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n").unwrap();

    let mut response = String::new();
    tls.read_to_string(&mut response).unwrap();
    print("Response: {}\n", response);
}
```

### TLS Server (serve HTTPS)

```zeta
use @crypto/ztls::{ServerConfig, ServerConnection, Stream};
use @crypto/ztls::crypto::ring::default_provider;
use std::sync::Arc;
use std::net::TcpListener;

fn main() {
    // Load certificate chain and private key
    let cert_chain = load_certificates("cert.pem").unwrap();
    let key = load_private_key("key.pem").unwrap();

    let config = ServerConfig::builder_with_provider(default_provider().into())
        .with_safe_defaults()
        .with_no_client_auth()
        .with_single_cert(cert_chain, key)
        .unwrap();

    let listener = TcpListener::bind("0.0.0.0:4433").unwrap();

    for stream in listener.incoming() {
        let tcp = stream.unwrap();
        let mut server = ServerConnection::new(Arc::new(config)).unwrap();
        let mut tls = Stream::new(&mut server, tcp);

        // Read HTTP request over TLS
        let mut request = String::new();
        tls.read_to_string(&mut request).unwrap();
        tls.write_all(b"HTTP/1.1 200 OK\r\nContent-Length: 12\r\n\r\nHello Zeta!\n").unwrap();
    }
}
```

## Crypto Providers

ZTLS supports pluggable cryptographic backends:

```zeta
use @crypto/ztls::crypto::ring::default_provider;  // Default (ring-compatible)
use @crypto/ztls::crypto::aws_lc_rs::default_provider;  // AWS-LC (FIPS capable)
```

## TLS Versions

```zeta
use @crypto/ztls::ProtocolVersion;

fn main() {
    // Enable only TLS 1.3
    let config = ClientConfig::builder_with_provider(default_provider().into())
        .with_safe_defaults()
        .with_protocol_versions(&[ProtocolVersion::TLSv1_3])
        .with_root_certificates(root_store)
        .with_no_client_auth();
}
```

## API Overview

| Function / Type | Description |
|----------------|-------------|
| `ClientConfig` | TLS client configuration (certificates, cipher suites, versions) |
| `ServerConfig` | TLS server configuration |
| `ClientConnection` | Established TLS client session |
| `ServerConnection` | Established TLS server session |
| `Stream` | Bidirectional TLS stream wrapping a TCP socket |
| `ServerName` | DNS name for SNI and certificate verification |
| `CryptoProvider` | Pluggable cryptographic backend |
| `RootCertStore` | Trusted certificate authority store |
| `Certificate` | X.509 certificate |
| `PrivateKey` | Private key (RSA, ECDSA, Ed25519) |
| `Tls12` / `Tls13` | Protocol version-specific state machines |

## Features

- **TLS 1.2** and **TLS 1.3** — both client and server
- **Multiple crypto backends** — ring-compatible default, AWS-LC for FIPS
- **Certificate validation** — webpki-based chain verification, CRL support
- **SNI** — Server Name Indication for virtual hosting
- **Session resumption** — faster reconnects
- **ALPN** — Application-Layer Protocol Negotiation (HTTP/2, etc.)
- **QUIC integration** — TLS 1.3 for QUIC transport
- **Post-quantum key exchange** — ML-KEM (Kyber) via AWS-LC backend

## Tips

- Always use `with_safe_defaults()` to get a secure baseline
- For production, load root certificates from the system bundle or webpki-roots
- Session resumption dramatically reduces handshake latency on reconnects
- Use `Stream` for simple read/write; use `split()` for separate read/write halves
- Test TLS configurations with badssl.com or test vectors before deploying

## License

MIT OR Apache-2.0
