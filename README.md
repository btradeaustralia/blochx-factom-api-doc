# BlochX Factom API

The API is currently hosted at [`http://blochx-factom-api.bittradelabs.com`](`http://blochx-factom-api.bittradelabs.com`)

### HMAC Authentication

This API uses [HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code)
based authentication.

HMAC authentication is applied as follows:

Take a request of JSON format, e.g:

```json
{
    "to_publish": {"a json": "document"},
    "chain_id": "00000000...."    // optional
}
```

Serialize that request e.g. in JavaScript

`var serd_data = '{"to_publish":{"a json":"document"}, "chain_id": "000000...."}';`

Construct the HMAC over `serd_data` and use it as value for the `payload` field, as a string:

```json
{
    "payload": "{\"to_publish\":{\"a json\":\"document\"}, \"chain_id\": \"000000....\"}",
    "hmac": "1d55296fe49dd34b6889e6bf9a8a1a772c1b4ff291872204bdfd4b8698425428",
    "user_id": "ec08708e50a7b5af5eebba66a8793c693d631fd9659c658c4100057ae8151268"  // optional, not currently used
}
```

See the section below for ways to experiment with HMAC without any programming
but using command line and online tools instead.

## Using curl and online HMAC generator to experiment

Before you write your API (in your programming language of choice) you can play with
this endpoint using the command line tool: `curl`

You would send the POST request above using (all on one line)

    curl --data-binary '{ "payload": "{\"final hmac test\": \"passed!\"}",     "hmac":"1d55296fe49dd34b6889e6bf9a8a1a772c1b4ff291872204bdfd4b8698425428" }' http://blochx-factom-api.bittradelabs.com/api0/publish_data

But how did we get this HMAC in the first place? You can use [`http://www.freeformatter.com/hmac-generator.html`](http://www.freeformatter.com/hmac-generator.html) . Make sure you select SHA256 from the dropdown and use your Secret Key.

You need to hash the payload, however there is a catch. Simply copying and
pasting the payload above will not work. Why? Because the `\"` are what are
called escape sequences. The only reason they appear at all in the Sample Post
Request and curl command line above is because JSON uses double quotes (`"`)
itself. So how do you represent double quotes inside an encoding that uses
double quotes? Simple: you distinguish them with a slash in front of them.

However, the real string that you are HMACing is as follows (spaces important!)

    {"final hmac test": "passed!"}

(Essentially you just replace `\"` with `"` everywhere)

If you HMAC that in the online HMAC page (`http://www.freeformatter.com/hmac-
generator.html`) you should get:

    1d55296fe49dd34b6889e6bf9a8a1a772c1b4ff291872204bdfd4b8698425428

You won't have to worry about escape sequences explicitly in your programming
language of choice. It should handle escaping for you. But when you're trying
stuff out manually you'll need to be aware of this.

## API Endpoints

### GET `/api0/info`

Sample Response:

```json
{
    "verify_key": "0c5e15360f57c16e8d7ed35a3f34da0e9d0cb1ecd7027f458292b096eac41aab",
    "chain_id": "ec08708e50a7b5af5eebba66a8793c693d631fd9659c658c4100057ae8151268",
    "height": 57351,
    "status": "running"
}
```

Gives some super basic info, just to confirm that we're running and can read factomd, as well as the chain_id and the verification key for all signed data.

### POST `/api0/publish_data`

Post payload should be either

1. a JSON object
2. a JSON array containing JSON objects

In case 1 a single entry will be published to a Factom chain.
In case 2 multiple single entries will be published to a Factom chain, one
for each object in the JSON array.

This request is authenticated with an HMAC and a shared key (for now, later on shared keys can be user specific by including a userid).

**Sample Post Request**

The following POST request will publish the following to Factom Blockchain.

    { "final hmac test" : "passed!" }

First you create the right JSON object for the payload

    { "to_publish" : { "final hmac test":"passed!" } }

And then serialize it to

    "{\"to_publish\":{\"final hmac test\":\"passed!\"}}"

(this particular serialization removed all the spaces but it doesn't have to,
the important thing is that the HMAC is done on the _serialized_ string)

Here is the body of the POST request:

```json
{
    "payload": "{\"to_publish\":{\"final hmac test\":\"passed!\"}}",
    "hmac": "b6bdfedf93ea154398753711016f0c799fa7aaa513c52156810daaa84ff77110"
}
```

`hmac` should be a hex encoded sha256 hmac using the shared key.

`payload` should be a _serialized_ json payload. The payload needs to be serialized because serialization and deserialization of JSON is not deterministic.
For the HMAC to work both ends have to operate over the _exact_ same payload, and the only way to guarantee that is to operate over JSON serialized on the client end.

Payload format:

NB: The `chain_id` field is optional; will use the default chain if this param is not provided.

```json
{
    "to_publish": {"some data": "in unserialized json"},
    "chain_id": "000000...."   // optional
}
```


Response Sample:

```json
{
    "result": "120667a891eee9af6327e8a32e234b7ff3284f785f4780b67241902ab646f2f8"
}
```

The `result` field contains the hash of the newly created entry.

### GET `/api0/get_all_signed_data`

Sample Response:

```json
{
    "results": [
        ["{\"msg\": \"A second sample message\"}", "120667a891eee9af6327e8a32e234b7ff3284f785f4780b67241902ab646f2f8"],
        ["{\"msg\": \"an initial message, 1035 5th Oct 2016\"}", "3e1d01618cb59afd98109110d120763e93251ce162f736f243325d3543f52271"]
    ],
    "okay": true
}
```

Note that `results` is a list of list of (serialized JSON, hexlified hash) pairs. Serialization is because what's stored on the chain is binary data, and it's then up to the next layer to determine if that msg was valid or not (for whatever purposes).

While it's _technically_ possible that non JSON data can be committed to the chain, the `publish_data` handler currently expects the request body to be a JSON object which is then serialized and chucked on the chain, so this particular API won't publish non-json data.

### GET `/api0/get_all_signed_data/<chain_id_hex>`

As above, except that the data for the provided `chain_id` will be returned. Note the chain id must be hexlified.

### POST `/api0/new_chain`

This call will create a new chain. It should be authenticated via hmac.

Format of _payload_: (to be serialized and signed via HMAC)

```json
{
    "name": "local-chain-descriptor"
}
```

Format of _request_: (what is actually sent to the server)

```json
{
    "payload": "{\"name\":\"local-chain-descriptor\"}",
    "hmac": "1d55296fe49dd34b6889e6bf9a8a1a772c1b4ff291872204bdfd4b8698425428"
}
```

Sample Response:

```json
{ "okay": true,
  "chain_id": "ec08708e50a7b5af5eebba66a8793c693d631fd9659c658c4100057ae8151268",
  "name": "local-chain-descriptor"
}
```


## API Changes History

* 2016-10-11
  * `/info` field `chain_id` changed to `default_chain_id`
  * `/publish_data` payload changed from serialized json (`serd_json`) to `{ 'to_publish': <serd_json>, 'chain_id': '012345...' }`. Note `chain_id` is optional.