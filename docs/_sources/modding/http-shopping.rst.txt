=================
HTTP Shopping API
=================

The HTTP Shopping API allows an external user with a blessed key to query the world servers directly without being signed into the game. The requests are returned in **binary** to not stall the real-time servers. Each world has a unique base API URL that can be requested from the universe discovery server. The API URLs will rarely change and are safe to cache for 24 hours. The HTTP Shopping route should be appended to the world API URLs to request the item listings per world.

How to query the per world base API URLs
----------------------------------------

* Request the world server listing via:

  * Creative: ``http://127.0.0.1:8950/list-gameservers``
  * Testing: ``https://ds-testing.playboundless.com:8902/list-gameservers``
  * Live: ``https://ds.playboundless.com:8902/list-gameservers``

* From the response extract the ``"apiURL"`` per world. This is base route URL for all API calls.

.. note::

  Testing and Live require ``https`` secure connections, whilst Local-sandbox only requires ``http`` connections.

How to query the HTTP Shopping API
----------------------------------

* ``GET`` request route is: ``/api/shopping/:type/:itemId``

  * ``:type`` of ``S`` for buy-from **shop-stands**,
  * ``:type`` of ``B`` for sell-to **request-baskets**,
  * ``:itemId`` for the item requested.

* Validation regex: ``"^/api/shopping/(B|S)/([0-9]+)$"``
* The request should include a header with your API key: ``Boundless-API-Key``. There are different keys for the Testing and Live servers.
* The response is a little-endian binary object.

Rate Limiting
-------------

* Requests which do not hit the cache and reach the game server are rate limited to balance across users and across the in-client shopping queuries made from Knowldege; this rate-limiting has only one rule that users should follow:
  * Each API key is permitted to have 1 request in-flight that hits a given game server: any attempt to run concurrent requests will return ``429`` responses.
* Any other request will either return an error state due to invalid arguments or server-state (starting-up, shutting-down, locked, not a public sovereign world..), or will return a ``200`` with the full response data.
* Rate limiting beyond the strict one-in-flight request will generally not apply unless there are many users (thousands) hitting the http endpoint concurrently; rate-limiting is handled by the server queueing incoming responses so that your response simply takes longer to come back if serving higher-priority users first - there is no need to handle delaying requests or retrying on the user-side.
* The shopping responses are also cached so the response may come back directly from the cache and not hit the game server at all further reducing the effects of rate-limiting.
* Responses will be returned up to a rate of 1 response per second for each api-key used to further ensure no hit to server performance, but you do not need to delay your next request.

How to decode the binary response
---------------------------------

.. include:: ../../client/scripts/shoppingdatatypes.cpp
  :start-after: {{Sphix_PublicQueryPack_Start}}
  :end-before: {{Sphix_PublicQueryPack_End}}

Example Requests
----------------

.. code-block:: bash

  # ItemID of ITEM_ROUGH_OORTSTONE is 32805

  # For testing against **Testing**.
  # Request the list of worlds to extract the "apiURL" for each world.
  curl https://ds-testing.playboundless.com:8902/list-gameservers

  # For "Havran I" the apiURL = https://gs-testing-euc1.playboundless.com/13/api
  curl -H "Boundless-API-Key: TESTING_KEY" https://gs-testing-euc1.playboundless.com/13/api/shopping/B/32805 > testing.euc1_t0_0.b.32805.bin
  curl -H "Boundless-API-Key: TESTING_KEY" https://gs-testing-euc1.playboundless.com/13/api/shopping/S/32805 > testing.euc1_t0_0.s.32805.bin

  # For running against **Live**.
  # Request the list of worlds to extract the "apiURL" for each world.
  curl https://ds.playboundless.com:8902/list-gameservers

  # For "Sochaltin I" the apiURL = https://gs-live-euc1.playboundless.com/1/api
  curl -H "Boundless-API-Key: LIVE_KEY" https://gs-live-euc1.playboundless.com/1/api/shopping/B/32805 > live.euc1_t0_0.b.32805.bin
  curl -H "Boundless-API-Key: LIVE_KEY" https://gs-live-euc1.playboundless.com/1/api/shopping/S/32805 > live.euc1_t0_0.s.32805.bin

Example Response Parsing
------------------------

Below is an example Python 3 script that will parse the example ``curl`` responses above. Note the highlighted section which incrementally parses the response per shop advert.

.. literalinclude:: ../../tools/dumpshopbin/dumpshopbin.py
  :language: python
  :linenos:
  :emphasize-lines: 16-23
