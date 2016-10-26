## API

### HMAC Authentication

HMAC authentication is applied as follows:

Take a request of JSON format, eg:

```json
{
    "to_publish": {"a json": "document"},
    "chain_id": "00000000...."    // optional
}
```

Serialized that request: `const serd_data = '{"to_publish":{"a json":"document"}, "chain_id": "000000...."}';`

Construct the hmac over `serd_data` and use it as value for the `payload` field, as a string:

```json
{
    "payload": "{\"to_publish\":{\"a json\":\"document\"}, \"chain_id\": \"000000....\"}",
    "hmac": "1d55296fe49dd34b6889e6bf9a8a1a772c1b4ff291872204bdfd4b8698425428",
    "user_id": "ec08708e50a7b5af5eebba66a8793c693d631fd9659c658c4100057ae8151268"  // optional, not currently used
}
```

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

Post payload should be JSON.

This request is authenticated with an HMAC and a shared key (for now, later on shared keys can be user specific by including a userid).

Sample Post Request:

```json
{
    "payload": "{\"to_publish\": \"{\"final hmac test\": \"passed!\"}\"}",
    "hmac": "1d55296fe49dd34b6889e6bf9a8a1a772c1b4ff291872204bdfd4b8698425428"
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




## API Changes History

* 2016-10-11
  * `/info` field `chain_id` changed to `default_chain_id`
  * `/publish_data` payload changed from serialized json (`serd_json`) to `{ 'to_publish': <serd_json>, 'chain_id': '012345...' }`. Note `chain_id` is optional.
