=================
HTTP LOD0 Map API
=================

The LOD0 Map HTTP API allows an external user with a blessed key to query the world servers directly to generate an RGB image of the planet at a resolution of 1 pixel per block. The requests are returned as a GZIP compressed TGA image file. Each world has a unique base API URL that can be requested from the universe discovery server. The API URLs will rarely change and are safe to cache for 24 hours. The HTTP LOD0 Map route should be appended to the world API URLs to request the map per world.

How to query the per world base API URLs
----------------------------------------

* See :ref:`modding/http-shopping:How to query the per world base API URLs`

How to query the HTTP LOD0 Map API
----------------------------------

* ``GET`` request route is: ``/api/lod0``

* The request should include a header with your API key: ``Boundless-API-Key``. There are different keys for the Testing and Live servers.
* The response is streamed back with ``Chunked Transfer Encoding`` returning a ``GZIP`` encoded ``TGA`` image which you may decompress (``gunzip`` on Linux/macOS or some other utility on Windows OS) and then convert to the image type of your choosing (We recommend PNG).
* The response purposefully takes quite a long time (9 to 10 minutes for a standard sized world) to ensure that the request cannot cause any performance problems to the world server.
* The returned map may not be fully consistent if there are users on the world modifying the world during the response processing: if someone blows up a bridge you may see half of the bridge in the map and half not in the map if the destruction occurred in the middle of processing that region of the world.

Error Codes
-----------

* a **403** status will be returned if the API key was not acceptable.
* a **503** status will be returned if the server is not in an acceptable state (Is starting up, is shutting down or being terminated, or is in a locked state for sovereign/creative worlds).
* a **503** status will also be returned if another request for the LOD0 Map is ongoing; in this case the response will include a ``Retry-After`` header with an approximate time until the API will be available to use again; there is no enforced fair-use of the endpoint though if querying the Boundless MMO world servers you should not expect to see this kind of response as the map will be cached at a higher level so that you simply get the older version of the map in the meantime instead.

Example Requests
----------------

.. code-block:: bash

  # request the map and ask curl to decompress
  curl -H "Boundless-API-Key: TESTING_KEY" https://gs-testing-euc1.playboundless.com/13/api/lod0 --output testing.euc1_t0_0.map.tga.gz

  # convert to a png file with ImageMagick
  convert testing.euc1_t0_0.map.tga.gz testing.euc1_t0_0.map.png
