## Takeaways
Out of rustls, openssl, and s2n-tls, s2n-tls is the least-syscall efficient and the least record-efficient. 

Performing a handshake and then exchanging one byte of application data between a client and server requires the following number of IO calls
- rustls: 20
- s2n-tls (recv buffering on): 21
- s2n-tls (default): 41
- openssl (default): 41
> Note that openssl sends session tickets by default but s2n-tls does not, so s2n-tls is strictly less syscall-efficient than openssl.

The syscall differences are explained by the following behaviors

|                        | read buffering | write buffering | record coalescing |
| ---------------------- | -------------- | --------------- | ----------------- |
| rustls                 | yes            | no              | yes               |
| s2n-tls                | no             | no              | no                |
| s2n-tls (w/ buffering) | yes            | no              | no                |
| openssl                | no             | yes             | no                |
### read buffering
Read buffering is a feature where the TLS implementation will try to read as much data as possible to fill it's IO buffer. 

If a TLS implementation doesn't implement read buffering, then it will read exactly the size of each record. This requires the TLS implementation to perform 2 read calls per record. The first read call gets the record header which contains the record length, and then the second read call attempts to read exactly the record length.

rustls is the only implementation which enables read buffering by default. s2n-tls does offer read buffering, but it is disabled by default.
### write buffering
Write buffering is a feature where a TLS implementation which needs to write message A, B, and C, will first store them in a contiguous chunk of memory `[A, B, C]` and then call `write` with that single chunk of memory.

If a TLS implementation doesn't implement write buffering, it will issue three separate `write` calls.

s2n-tls does not implement write buffering.
### record coalescing
Record coalescing is a feature where multiple TLS messages can be combined into a single record. For example the `EncryptedExtensions`, `Certificate`, `CertVerify`, and `Finished` messages are all encrypted under the same keys. Instead of writing 4 different records for the 4 different messages, the TLS implementation can write a single record that contains 4 different mesages.

s2n-tls does not implement record coalescing. 




# Appendix
This appendix shows the read and write behavior of an client and server from a specific implementation doing a handshake with each other.
## Rustls

```
running 1 test
0xf68e9534da28 wrote Ok(1452)
0xf68e9534da28 trying to read into buffer of length 4096
        Ok(0)
0xf68e9534ded8 trying to read into buffer of length 4096
        Ok(1452)
0xf68e9534ded8 wrote Ok(1215)
0xf68e9534ded8 wrote Ok(6)
0xf68e9534ded8 wrote Ok(2809)
0xf68e9534ded8 trying to read into buffer of length 4096
        Ok(0)
0xf68e9534da28 trying to read into buffer of length 4096
        Ok(4030)
0xf68e9534da28 wrote Ok(6)
0xf68e9534da28 wrote Ok(74)
0xf68e9534ded8 trying to read into buffer of length 4096
        Ok(80)
0xf68e9534ded8 wrote Ok(184)
0xf68e9534da28 wrote Ok(23)
0xf68e9534ded8 trying to read into buffer of length 4096
        Ok(23)
0xf68e9534ded8 wrote Ok(23)
0xf68e9534da28 trying to read into buffer of length 4096
        Ok(207)
0xf68e9534da28 wrote Ok(24)
0xf68e9534ded8 wrote Ok(24)
0xf68e9534da28 trying to read into buffer of length 4096
        Ok(24)
0xf68e9534ded8 trying to read into buffer of length 4096
        Ok(24)
total io calls: 20
test cohort::rustls::tests::handshake ... ok
```

## s2n-tls (no recv buffering)

```
0xed17680364b0 wrote Ok(1521)
0xed17680364b0 trying to read into buffer of length 5
        Ok(0)
0xed1768047e40 trying to read into buffer of length 5
        Ok(5)
0xed1768047e40 trying to read into buffer of length 1516
        Ok(1516)
0xed1768047e40 wrote Ok(1215)
0xed1768047e40 wrote Ok(6)
0xed1768047e40 wrote Ok(28)
0xed1768047e40 wrote Ok(841)
0xed1768047e40 wrote Ok(286)
0xed1768047e40 wrote Ok(58)
0xed1768047e40 trying to read into buffer of length 5
        Ok(0)
0xed17680364b0 trying to read into buffer of length 5
        Ok(5)
0xed17680364b0 trying to read into buffer of length 1210
        Ok(1210)
0xed17680364b0 trying to read into buffer of length 5
        Ok(5)
0xed17680364b0 trying to read into buffer of length 1
        Ok(1)
0xed17680364b0 trying to read into buffer of length 5
        Ok(5)
0xed17680364b0 trying to read into buffer of length 23
        Ok(23)
0xed17680364b0 trying to read into buffer of length 5
        Ok(5)
0xed17680364b0 trying to read into buffer of length 836
        Ok(836)
0xed17680364b0 trying to read into buffer of length 5
        Ok(5)
0xed17680364b0 trying to read into buffer of length 281
        Ok(281)
0xed17680364b0 trying to read into buffer of length 5
        Ok(5)
0xed17680364b0 trying to read into buffer of length 53
        Ok(53)
0xed17680364b0 wrote Ok(6)
0xed17680364b0 wrote Ok(58)
0xed1768047e40 trying to read into buffer of length 5
        Ok(5)
0xed1768047e40 trying to read into buffer of length 1
        Ok(1)
0xed1768047e40 trying to read into buffer of length 5
        Ok(5)
0xed1768047e40 trying to read into buffer of length 53
        Ok(53)
0xed17680364b0 wrote Ok(23)
0xed1768047e40 trying to read into buffer of length 5
        Ok(5)
0xed1768047e40 trying to read into buffer of length 18
        Ok(18)
0xed1768047e40 wrote Ok(23)
0xed17680364b0 trying to read into buffer of length 5
        Ok(5)
0xed17680364b0 trying to read into buffer of length 18
        Ok(18)
0xed17680364b0 wrote Ok(24)
0xed1768047e40 wrote Ok(24)
0xed17680364b0 trying to read into buffer of length 5
        Ok(5)
0xed17680364b0 trying to read into buffer of length 19
        Ok(19)
0xed1768047e40 trying to read into buffer of length 5
        Ok(5)
0xed1768047e40 trying to read into buffer of length 19
        Ok(19)
total io calls: 41
test cohort::s2n_tls::tests::handshake ... ok
```

## s2n-tls w/ recv buffering
```
0xe63d140364b0 wrote Ok(1521)
0xe63d140364b0 trying to read into buffer of length 16384
        Ok(0)
0xe63d14047e40 trying to read into buffer of length 16384
        Ok(1521)
0xe63d14047e40 wrote Ok(1215)
0xe63d14047e40 wrote Ok(6)
0xe63d14047e40 wrote Ok(28)
0xe63d14047e40 wrote Ok(841)
0xe63d14047e40 wrote Ok(286)
0xe63d14047e40 wrote Ok(58)
0xe63d14047e40 trying to read into buffer of length 16384
        Ok(0)
0xe63d140364b0 trying to read into buffer of length 16384
        Ok(2434)
0xe63d140364b0 wrote Ok(6)
0xe63d140364b0 wrote Ok(58)
0xe63d14047e40 trying to read into buffer of length 16384
        Ok(64)
0xe63d140364b0 wrote Ok(23)
0xe63d14047e40 trying to read into buffer of length 16384
        Ok(23)
0xe63d14047e40 wrote Ok(23)
0xe63d140364b0 trying to read into buffer of length 16384
        Ok(23)
0xe63d140364b0 wrote Ok(24)
0xe63d14047e40 wrote Ok(24)
0xe63d140364b0 trying to read into buffer of length 16384
        Ok(24)
0xe63d14047e40 trying to read into buffer of length 16384
        Ok(24)
total io calls: 22
test cohort::s2n_tls::tests::handshake ... ok
```
## openssl

```
0xe9039c091380 wrote Ok(1531)
0xe9039c091380 trying to read into buffer of length 5
        Ok(0)
0xe9039c081b00 trying to read into buffer of length 5
        Ok(5)
0xe9039c081b00 trying to read into buffer of length 1526
        Ok(1526)
0xe9039c081b00 wrote Ok(4092)
0xe9039c081b00 trying to read into buffer of length 5
        Ok(0)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 1210
        Ok(1210)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 1
        Ok(1)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 23
        Ok(23)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 2478
        Ok(2478)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 281
        Ok(281)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 69
        Ok(69)
0xe9039c091380 wrote Ok(80)
0xe9039c081b00 trying to read into buffer of length 5
        Ok(5)
0xe9039c081b00 trying to read into buffer of length 1
        Ok(1)
0xe9039c081b00 trying to read into buffer of length 5
        Ok(5)
0xe9039c081b00 trying to read into buffer of length 69
        Ok(69)
0xe9039c081b00 wrote Ok(255)
0xe9039c081b00 wrote Ok(255)
0xe9039c091380 wrote Ok(23)
0xe9039c081b00 trying to read into buffer of length 5
        Ok(5)
0xe9039c081b00 trying to read into buffer of length 18
        Ok(18)
0xe9039c081b00 wrote Ok(23)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 250
        Ok(250)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 250
        Ok(250)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 18
        Ok(18)
0xe9039c091380 wrote Ok(24)
0xe9039c081b00 wrote Ok(24)
0xe9039c091380 trying to read into buffer of length 5
        Ok(5)
0xe9039c091380 trying to read into buffer of length 19
        Ok(19)
0xe9039c081b00 trying to read into buffer of length 5
        Ok(5)
0xe9039c081b00 trying to read into buffer of length 19
        Ok(19)
total io calls: 41
test cohort::openssl::tests::handshake ... ok
```