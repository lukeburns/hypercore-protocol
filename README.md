# peer-protocol

Stream protocol that implements multiple channels and extensible messaging. A stripped down version of [hypercore-protocol](https://github.com/mafintosh/hypercore-protocol).

```
npm install peer-protocol
```

## Usage

``` js
var protocol = require('peer-protocol')

var stream = protocol()

// open a channel specified by a 32 byte key
var channel = stream.open(Buffer('deadbeefdeadbeefdeadbeefdeadbeef'))

stream.pipe(otherStream).pipe(stream)
```

## Extension API

#### `protocol = protocol.use(extensionName)`

Use an extension specified by the string name you pass in. Returns a new prototype

Will create a new method on all your channel objects that has the same name as the extension and emit an event with the same name when an extension message is received

``` js
protocol = protocol.use('ping')

var p = protocol()
var channel = p.open(someKey)

channel.on('handshake', function () {
  channel.on('ping', function (message) {
    console.log('received ping message', message)
  })

  channel.ping(Buffer('this is a ping message!'))
})
```

Per default all messages are buffers. If you want to encode/decode your messages you can specify an [abstract-encoding](https://github.com/mafintosh/abstract-encoding) compliant encoder as well

``` js
protocol = protocol.use({
  ping: someEncoder
})
```

## API

#### `var k = protocol.parseDiscoveryKey(buf)`

Parse the discovery key encoded in `buf` which is the first varint message taken from a hypercore feed stream.

#### `var p = protocol([options], [onopen])`

Create a new protocol instance. The returned object is a duplex stream
that you should pipe to another protocol instance over a stream based transport

If the remote peer joins a channel you haven't opened, hypercore will call an optional `onopen`
method if you specify it with the discovery key for that channel.

``` js
var p = protocol(function (discoveryKey) {
  // remote peer joined discoveryKey but you haven't
  // you can open the channel now if you want to join the channel

  // open with corresponding key to join
  var channel = p.open(Buffer('deadbeefdeadbeefdeadbeefdeadbeef'))
})
```

See below for more information about channels, keys, and discovery keys.
Other options include:

``` js
{
  id: optionalPeerId, // you can use this to detect if you connect to yourself
  encrypt: true // set to false to disable encryption for debugging purposes
}
```

If you don't specify a peer id a random 32 byte will be used.
You can access the peer id using `p.id` and the remote peer id using `p.remoteId`.

#### `var channel = p.open(key, [options])`

Open a stream channel. A channel uses the [sodium](https://github.com/mafintosh/sodium-prebuilt) module to encrypt all messages using the key you specify. The discovery key for the channel is send unencrypted together with a random 24 byte nonce. If you do not specify a discovery key in the options map, an HMAC of the string `hypercore` using the key as the password will be used.

#### `p.on('handshake')`

Emitted when a protocol handshake has been received. Afterwards you can check `.remoteId` to get the remote peer id.

#### `p.setTimeout(ms, [ontimeout])`

Will call the timeout function if the remote peer hasn't send any messages within `ms`. Will also send a heartbeat message to the other peer if you've been inactive for `ms / 2`

## Channel API

#### `channel[extension](data)`

Encodes data with the `extension` encoder and sends encoded data.

#### `channel.on(extension, (data) => { ... })`

Received data is decoded with `extension` decoder and an event is emitted with the decoded data.

#### `var bool = p.remoteSupports(extensionName)`

After the protocol instance emits `handshake` you can call this method to check
if the remote peer also supports one of your extensions.

## License

MIT
