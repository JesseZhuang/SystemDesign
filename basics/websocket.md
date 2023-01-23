# About

WebSocket is a computer communications protocol, providing full-duplex communication channels over a single TCP connection. The WebSocket protocol was standardized by the IETF as RFC 6455 in 2011. The current API specification allowing web applications to use this protocol is known as WebSockets.[1] It is a living standard maintained by the WHATWG and a successor to The WebSocket API from the W3C.[2]

# Websocket vs HTTP

HTTP is a stateless protocol that runs on top of TCP which is a connection-oriented protocol it guarantees the delivery of data packet transfer using the three-way handshaking methods and re-transmits the lost packets. 

-|HTTP|Websocket
-|-|-
duplex|half|full
messaging pattern|request-response|bi-directional
service push|not natively supported, client polling or streaming download techniques used|core feature
overhead|moderate per request/connection|moderate overhead to establish&maintain the connection, minimum per message
edge caching|core feature|not possible
supported clients|broad support|modern languages & clients



## References

[1]: https://websockets.spec.whatwg.org/
[2]: https://www.w3.org/TR/2021/NOTE-websockets-20210128/Overview.html

1. https://en.wikipedia.org/wiki/WebSocket
1. 