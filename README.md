jsfs
====

A general-purpose, deduplicating filesystem with a REST interface, jsfs is intended to provide low-level filesystem functions for Javascript applications.  Additional functionality (private file indexes, token lockers, centralized authentication, etc.) are deliberately avoided here and will be implemented in a modular fashion on top of jsfs.

#STATUS
JSFS 3.x introduces breaking changes to the REST API compared to earlier versions as well as the on-disk components (blocks, metadata, etc.).  I'll be releasing an upgrade tool at some point to make migration from 2.x JSFS systems easier but for now be aware that there's not a direct upgrade path at this time.

#REQUIREMENTS
* Node.js

#CONFIGURATION
*  Clone this repository
*  Copy config.ex to config.js
*  Create the `blocks` directory (`mkdir blocks`)
*  Start the server (`node server.js`, `npm start`,  foreman, pm2, etc.)

If you don't like storing the data in the same directory as the code (smart), edit config.js to change the location where data (blocks) are stored and restart jsfs for the changes to take effect.

JSFS can now store blocks across physical disk boundaries which is useful when you need to store more data than a single disk can hold.  At the moment JSFS simply distributes these blocks as evenly as possible across all configured storage devices.  There are thoughts about supporting configurations that provide redundancy through the use of multiple storage devices but for now you'll want to make sure the devices have their own redundancy (or a good backup) as loosing a storage device will cause data loss just like it would with a single device.

In addition to multiple locations, you specify the maximum amount of data that can be stored per-location.  The default config sets this very low (1024 bytes) so you can see what happens when you run out of space and respond accordingly.  Useage and capacity statistics are logged to the console periodically so you can keep an eye on usage before you hit the limit.
 
#API

##Keys and Tokens
Keys are used to unlock all operations that can be performed on an object stored in JSFS, and objects can have only one key.  With an `access_key`, you can execute all supported HTTP verbs (GET, PUT, DELETE) as well as generate tokens that grant varying degrees of access to the object.

Tokens are more ephemeral, and any number of them can be generated to grant varying degrees of access to an object.  Token generation is described later.

##Parameters and Headers
jsfs uses several parameters to control access to objects and how they are stored.  These values can also be supplied as request headers by adding a leading "x-" and changing "_" to "-" (`access_token` becomes `x-access-token`). Headers are preferred to querystring parameters because they are less likely to collide but both function the same. 

###private
By default all objects stored in jsfs are public and will be accessible via `GET` request and show up in directory listings.  If the `private` parameter is set to `true` a valid `access_key` or `access_token` must be supplied to access the object.

*NOTE: as `private` objects are not included in directory listings it is up to the client to keep track of them and their associated keys.*
 
###encrypted
Don't use encryption for now, it needs more testing in light of recent changes.
Set this paramter to `true` to encrypt data before it is stored on disk.  Once enabled, decryption happens automatically on `GET` requests and additional modifications via `PUT` will be encrypted as well.

*NOTE: encryption increases CPU utilization and potentially reduces deduplication performance, so use only when necissary.*

###version
This parameter allows you to retreive a specific version of a file when making a `GET` request.  Each time a `PUT` request is made to an existing URL a new version is created and an `x-version` header is returned if the `PUT` request is sucessful.  A list of versions can be displayed by appending a `/` to the end of the URL for a file, which will return a JSON array of versions for the specified file.

###access_key
Specifying a valid access_key authorizes the use of all supported HTTP verbs and is required for requests to change the `access_key` or generate `access_token`s.  When a new object is stored jsfs will generate an `access_key` automatically and return it in the response to a `POST` request.  Additionally the client can supply a custom `access_key` by supplying this parameter to the initial `POST` request.

An `access_key` can be changed by supplying the current `access_key` along with the `replacement_access_key` parameter.  This will cause any existing `access_token`s to become invalid.

*NOTE: changing the `access_key` of an encrypted object is currently unsupported!*.

###access_token
An `access_token` must be provided to execute any request on a `private` object, and is required for `PUT` and `DELETE` if an `access_key` is not supplied.

####Generating access_tokens
Currently there are two types of `access_token`s: durable and temporary.  Both are generated by creating a string that describes what access is granted and then using SHA1 to generate a hash of this string, but the format and use is a little different.

Durable `access_token`s are generated by concatinating an object's `access_key` with the HTTP verb that the token will be used for.

Example to grant GET access:

     "077785b5e45418cf6caabdd686719813fb22e3ce" + "GET"

This string is then hashed with SHA1 and can be used to perform a GET request for the object whose `access_key` was used to generate the token.

To make a temporary token for this same object, concatinate the `access_key` with the HTTP verb and the expiration time in epoc format (milliseconds since midnight, 01/01/1970):

     "077785b5e45418cf6caabdd686719813fb22e3ce" + "GET" + "1424877559581"

This string is then hashed with SHA1 and supplied as a parameter or header with the request, along with an additional parameter named `expires` which is set to match the expiration time used above.  When jsfs receives the request it generates the same token based on the stored `access_key`, the HTTP method of the incoming request and the supplied `expires` parameter to validate the `access_token`.

*NOTE: all `access_tokens` can be immediately invalidated by changing an objects `access_key`, however if individual `access_tokens` need to be invalidated a pattern of requesting new, temporary tokens before each request is recommended.*

##POST
Stores a new object at the specified URL.  If the object exists jsfs returns `405 method not allowed`.

###EXAMPLE
Request:

     curl -X POST --data-binary @Brinstar.mp3 "http://localhost:7302/music/Brinstar.mp3"

Response:
````
{
    "url": "/localhost/music/Brinstar.mp3",
    "created": 1424878242595,
    "version": 0,
    "private": false,
    "encrypted": false,
    "fingerprint": "fde752ca6541c16ec626a3cf6e45e835cfd9db9b",
    "access_key": "fde752ca6541c16ec626a3cf6e45e835cfd9db9b",
    "content_type": "application/x-www-form-urlencoded",
    "file_size": 7678080,
    "block_size": 1048576,
    "blocks": [
        {
            "block_hash": "610f0b4c20a47b4162edc224db602a040cc9d243",
            "last_seen": "./blocks/"
        },
        {
            "block_hash": "60a93a7c97fd94bb730516333f1469d101ae9d44",
            "last_seen": "./blocks/"
        },
        {
            "block_hash": "62774a105ffc5f57dcf14d44afcc8880ee2fff8c",
            "last_seen": "./blocks/"
        },
        {
            "block_hash": "14c9c748e3c67d8ec52cfc2e071bbe3126cd303a",
            "last_seen": "./blocks/"
        },
        {
            "block_hash": "8697c9ba80ef824de9b0e35ad6996edaa6cc50df",
            "last_seen": "./blocks/"
        },
        {
            "block_hash": "866581c2a452160748b84dcd33a2e56290f1b585",
            "last_seen": "./blocks/"
        },
        {
            "block_hash": "6c1527902e873054b36adf46278e9938e642721c",
            "last_seen": "./blocks/"
        },
        {
            "block_hash": "10938182cd5e714dacb893d6127f8ca89359fec7",
            "last_seen": "./blocks/"
        }
    ]
}
````

JSFS automatically "namespaces" URL's based on the host name used to make the request.  This makes it easy to create partitioned storage by pointing multiple hostnames at the same JSFS server.  This is accomplished by expanding the host in reverse notation (i.e.: `foo.com` becomes `.com.foo`); this is handled transparrently by JSFS from the client's perspective.

This means that you can point DNS records for `foo.com` and `bar.com` to the same JSFS server and then POST `http://foo.com:7302/files/baz.txt` and `http://bar.com:7302/files/baz.txt` without a conflict.

This also means that `GET http://foo.com:7302/files/baz.txt` and `GET http://bar.com:7302/files/baz.txt` do not return the same file, however if you need to access a file stored via a different host you can reach it using its absolute address (in this case, `http://bar.com:7302/.com.foo/files/baz.txt`).

##GET
Retreives the object stored at the specified URL.  If the file does not exist a `404 not found` is returned.  If the URL ends with a trailing slash `/` a directory listing of non-private files stored at the specified location.

###EXAMPLE
Request (directory):

     curl http://localhost:7302/music/

Response:

````
[
    {
        "url": "/localhost/music/Brinstar.mp3",
        "created": 1424878242595,
        "version": 0,
        "private": false,
        "encrypted": false,
        "fingerprint": "fde752ca6541c16ec626a3cf6e45e835cfd9db9b",
        "access_key": "fde752ca6541c16ec626a3cf6e45e835cfd9db9b",
        "content_type": "application/x-www-form-urlencoded",
        "file_size": 7678080,
        "block_size": 1048576,
        "blocks": [
            {
                "block_hash": "610f0b4c20a47b4162edc224db602a040cc9d243",
                "last_seen": "./blocks/"
            },
            {
                "block_hash": "60a93a7c97fd94bb730516333f1469d101ae9d44",
                "last_seen": "./blocks/"
            },
            {
                "block_hash": "62774a105ffc5f57dcf14d44afcc8880ee2fff8c",
                "last_seen": "./blocks/"
            },
            {
                "block_hash": "14c9c748e3c67d8ec52cfc2e071bbe3126cd303a",
                "last_seen": "./blocks/"
            },
            {
                "block_hash": "8697c9ba80ef824de9b0e35ad6996edaa6cc50df",
                "last_seen": "./blocks/"
            },
            {
                "block_hash": "866581c2a452160748b84dcd33a2e56290f1b585",
                "last_seen": "./blocks/"
            },
            {
                "block_hash": "6c1527902e873054b36adf46278e9938e642721c",
                "last_seen": "./blocks/"
            },
            {
                "block_hash": "10938182cd5e714dacb893d6127f8ca89359fec7",
                "last_seen": "./blocks/"
            }
        ]
    }
]
````

Request (file):

     curl -o Brinstar.mp3 http://localhost:730s/music/Brinstar.mp3

Response:
The binary file is stored in new local file called `Brinstar.mp3`.

##PUT
Updates an existing object stored at the specified location.  This method requires authorization, so requests must include a valid `x-access-key` or `x-access-token` header for the specific file, otherwise `401 unauthorized` will be returned.  If the file does not exist `405 method not allowed` is returned.

###EXAMPLE
Request:

     curl -X PUT -H "x-access-key: 7092bee1ac7d4a5c55cb5ff61043b89a6e32cf71"  --data-binary @Brinstar.mp3 "http://localhost:7302/music/Brinstar.mp3"

Result:
`HTTP 200`

##DELETE
Removes the file at the specified URL.  This method requires authorization so requests must include a valid `x-access-key` or `x-access-token` header for the specified file.  If the token is not supplied or is invalid `401 unauthorized` is returend.  If the file does not exist `405 method not allowed` is returned.

###Example
Request:

     curl -X DELETE -H "x-access-token: 7092bee1ac7d4a5c55cb5ff61043b89a6e32cf71" "http://localhost:7302/music/Brinstar.mp3"

Response
`HTTP 200` if sucessful.

##HEAD
Returns status and header information for the specified URL.

###Example
Request:

    curl -I "http://localhost:7302/music/Brinstar.mp3"

Response:
````
HTTP/1.1 200 OK
Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS
Access-Control-Allow-Headers: Accept,Accept-Version,Content-Type,Api-Version,Origin,X-Requested-With,Range,X_FILENAME,X-Access-Key,X-Replacement-Access-Key,X-Access-Token,X-Encrypted,X-Private
Access-Control-Allow-Origin: *
Content-Type: application/x-www-form-urlencoded
Content-Length: 7678080
Date: Wed, 25 Feb 2015 15:43:03 GMT
Connection: keep-alive
````

*NOTE: some response information above may be removed in later versions of jsfs, in particular the `blocks` section as it's not directly useful to clients.*
