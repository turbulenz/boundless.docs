================
HTTP Beacons API
================

The Beacons HTTP API allows an external user with a blessed key to query the world servers directly to query details of all beacons on the world and a "beacon map" of the world returning GZIP compressed little-endian binary packed data. Each world has a unique base API URL that can be requested from the universe discovery server. The API URLs will rarely change and are safe to cache for 24 hours. The HTTP Beacons route should be appended to the world API URLs to request the map per world.

How to query the per world base API URLs
----------------------------------------

* See :ref:`modding/http-shopping:How to query the per world base API URLs`

How to query the HTTP Beacons API
---------------------------------

* ``GET`` request route is: ``/api/beacons``

* The request should include a header with your API key: ``Boundless-API-Key``. There are different keys for the Testing and Live servers.
* The response is streamed back with ``Chunked Transfer Encoding`` returning a ``GZIP`` encoded binary data pack (``gunzip`` on Linux/macOS or some other utility on Windows OS).
* The response will run so as not to stall the server in any way, but will generally take less than 1 second to process.
* The returned beacon details may not be fully consistent if changes occur during processing but the data will be fully self consistent; it will not reference beacons whose details are not listed in the response.

Error Codes
-----------

* a **403** status will be returned if the API key was not acceptable.
* a **503** status will be returned if the server is not in an acceptable state (Is starting up, is shutting down or being terminated, or is in a locked state for sovereign/creative worlds).
* a **503** status will also be returned if another request for the beacons is ongoing; in this case the response will include a ``Retry-After`` header with an approximate time until the API will be available to use again; there is no enforced fair-use of the endpoint though if querying the Boundless MMO world servers you should not expect to see this kind of response as the data will be cached at a higher level so that you simply get the older version of the data in the meantime instead.

How to decode the decompressed binary response
----------------------------------------------

.. include:: ../../server/src/servermanager.cpp
  :start-after: {{Sphix_BeaconsPack_Start}}
  :end-before: {{Sphix_BeaconsPack_End}}

Example Requests
----------------

.. code-block:: bash

  # request the beacons data and ask curl to decompress it
  curl --compressed -H "Boundless-API-Key: TESTING_KEY" https://gs-testing-euc1.playboundless.com/13/api/beacons --output testing.euc1_t0_0.beacons.bin

Example Response Parsing
------------------------

Below is an example Python 3 script that will parse the decompressed beacons data response above.

.. literalinclude:: ../../tools/dumpbeaconsbin/dumpbeaconsbin.py
  :language: python
  :linenos:
