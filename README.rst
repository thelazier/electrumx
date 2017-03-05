.. image:: https://travis-ci.org/kyuupichan/electrumx.svg?branch=master
    :target: https://travis-ci.org/kyuupichan/electrumx
.. image:: https://coveralls.io/repos/github/kyuupichan/electrumx/badge.svg
    :target: https://coveralls.io/github/kyuupichan/electrumx

===============================================
ElectrumX - Reimplementation of electrum-server
===============================================

For a future network with bigger blocks.

  :Licence: MIT
  :Language: Python (>= 3.5.3)
  :Author: Neil Booth

Getting Started
===============

See `docs/HOWTO.rst`_.

Features
========

- Efficient, lightweight reimplementation of electrum-server
- Fast synchronization of bitcoin mainnet from Genesis.  Recent
  hardware should synchronize in well under 24 hours.  The fastest
  time to height 448k (mid January 2017) reported is under 4h 30m.  On
  the same hardware JElectrum would take around 4 days and
  electrum-server probably around 1 month.
- The full Electrum protocol is implemented.  The only exception is
  the blockchain.address.get_proof RPC call, which is not used by
  Electrum GUI clients, and can only be invoked from the command line.
- Various configurable means of controlling resource consumption and
  handling denial of service attacks.  These include maximum
  connection counts, subscription limits per-connection and across all
  connections, maximum response size, per-session bandwidth limits,
  and session timeouts.
- Minimal resource usage once caught up and serving clients; tracking the
  transaction mempool appears to be the most expensive part.
- Fully asynchronous processing of new blocks, mempool updates, and
  client requests.  Busy clients should not noticeably impede other
  clients' requests and notifications, nor the processing of incoming
  blocks and mempool updates.
- Daemon failover.  More than one daemon can be specified, and
  ElectrumX will failover round-robin style if the current one fails
  for any reason.
- Peer discovery protocol removes need for IRC
- Coin abstraction makes compatible altcoin and testnet support easy.

Motivation
==========

Mainly for privacy reasons, I have long wanted to run my own Electrum
server, but I struggled to set it up or get it to work on my
DragonFlyBSD system and lost interest for over a year.

In September 2016 I heard that electrum-server databases were getting
large (35-45GB when gzipped), and it would take several weeks to sync
from Genesis (and was sufficiently painful that no one seems to have
done it for about a year).  This made me curious about improvements
and after taking a look at the code I decided to try a different
approach.

I prefer Python3 over Python2, and the fact that Electrum is stuck on
Python2 has been frustrating for a while.  It's easier to change the
server to Python3 than the client, so I decided to write my effort in
Python3.

It also seemed like a good opportunity to learn about asyncio, a
wonderful and powerful feature introduced in Python 3.4.
Incidentally, asyncio would also make a much better way to implement
the Electrum client.

Finally though no fan of most altcoins I wanted to write a codebase
that could easily be reused for those alts that are reasonably
compatible with Bitcoin.  Such an abstraction is also useful for
testnets.

Implementation
==============

ElectrumX does not do any pruning or throwing away of history.  I want
to retain this property for as long as it is feasible, and it appears
efficiently achievable for the forseeable future with plain Python.

The following all play a part in making ElectrumX very efficient as a
Python blockchain indexer:

- aggressive caching and batching of DB writes
- more compact and efficient representation of UTXOs, address index,
  and history.  Electrum Server stores full transaction hash and
  height for each UTXO, and does the same in its pruned history.  In
  contrast ElectrumX just stores the transaction number in the linear
  history of transactions.  For at least another 5 years this
  transaction number will fit in a 4-byte integer, and when necessary
  expanding to 5 or 6 bytes is trivial.  ElectrumX can determine block
  height from a simple binary search of tx counts stored on disk.
  ElectrumX stores historical transaction hashes in a linear array on
  disk.
- placing static append-only metadata indexable by position on disk
  rather than in levelDB.  It would be nice to do this for histories
  but I cannot think of a way.
- avoiding unnecessary or redundant computations, such as converting
  address hashes to human-readable ASCII strings with expensive bignum
  arithmetic, and then back again.
- better choice of Python data structures giving lower memory usage as
  well as faster traversal
- leveraging asyncio for asynchronous prefetch of blocks to mostly
  eliminate CPU idling.  As a Python program ElectrumX is unavoidably
  single-threaded in its essence; we must keep that CPU core busy.

Python's ``asyncio`` means ElectrumX has no (direct) use for threads
and associated complications.


Roadmap
=======

- Python 3.6, which has several performance improvements relevant to
  ElectrumX
- UTXO root logic and implementation
- incremental history serving / pruning
- new features such as possibly adding label server functionality
- potentially move some functionality to C or C++


ChangeLog
=========

Version 1.0
-----------

* Minor doc tweaks only

Version 0.99.4
--------------

* Add support for Bitcoin Unlimited's nolnet; set **NET** to nolnet
* Choose 2 peers per bucket
* Minor bugfix

Version 0.99.3
--------------

* Require Python 3.5.3.  3.5.2 has asyncio API and socket-related issues.
  Resolves `#135`_
* Remove peer semaphore
* Improved Base58 handling for >1 byte version prefix (erasmospunk)

Version 0.99.2
--------------

* don't announce self if a non-public IP address
* logging tweaks

Version 0.99.1
--------------

* Add more verbose logging in attempt to understand issue `#135`_
* REPORT_TCP_PORT_TOR and REPORT_SSL_PORT_TOR were ignored when constructing
  IRC real names.  Fixes `#136`_
* Only serve chunk requests in forward direction; disconnect clients iterating
  backwards.  Minimizes bandwidth consumption caused by misbehaving Electrum
  clients.  Closes `#132`_
* Tor coin peers would always be scheduled for check, fixes `#138`_ (fr3aker)

Version 0.99
------------

Preparation for release of 1.0, which will only have bug fixes and
documentation updates.

* improve handling of daemon going down so that incoming connections
  are not blocked.  Also improve logging thereof.  Fixes `#100`_.
* add facility to disable peer discovery and/or self announcement,
  see `docs/ENVIRONMENT.rst`_.
* add FairCoin (thokon00)

Version 0.11.4
--------------

* peer handling fixes / improvements based on suggestions of hsmiths

Version 0.11.3
--------------

* fixed a typo in lib/peer.py pointed out by hsmiths

Version 0.11.2
--------------

* Preliminary implementation of script hash subscriptions to enable
  subscribing to updates of arbitrary scripts, not just those of
  standard bitcoin addresses.  I'll fully document once confirmed
  working as expected.
  Closes `#124`_.

Version 0.11.1
--------------

* report unconfirmed parent tx status correctly, and notify if that
  parent status changes.  Fixes `#129`_.

Version 0.11.0
--------------

* implementation of `docs/PEER_DISCOVERY.rst`_ for discovery of server
  peers without using IRC.  Closes `#104`_.  Since all testnet peers
  are ElectrumX servers, IRC advertising is now disabled on bitcoin
  testnet.

  Thanks to bauerj, hsmiths and JWU42 for their help testing these
  changes over the last month.
* you can now specify a tor proxy (or have it autodetected if local),
  and if an incoming connection seems to be from the proxy a
  tor-specific banner file is served.  See **TOR_BANNER_FILE** in
  `docs/ENVIRONMENT.rst`_.


**Neil Booth**  kyuupichan@gmail.com  https://github.com/kyuupichan

1BWwXJH3q6PRsizBkSGm2Uw4Sz1urZ5sCj


.. _#100: https://github.com/kyuupichan/electrumx/issues/100
.. _#104: https://github.com/kyuupichan/electrumx/issues/104
.. _#124: https://github.com/kyuupichan/electrumx/issues/124
.. _#129: https://github.com/kyuupichan/electrumx/issues/129
.. _#132: https://github.com/kyuupichan/electrumx/issues/132
.. _#135: https://github.com/kyuupichan/electrumx/issues/135
.. _#136: https://github.com/kyuupichan/electrumx/issues/136
.. _#138: https://github.com/kyuupichan/electrumx/issues/138
.. _docs/HOWTO.rst: https://github.com/kyuupichan/electrumx/blob/master/docs/HOWTO.rst
.. _docs/ENVIRONMENT.rst: https://github.com/kyuupichan/electrumx/blob/master/docs/ENVIRONMENT.rst
.. _docs/PEER_DISCOVERY.rst: https://github.com/kyuupichan/electrumx/blob/master/docs/PEER_DISCOVERY.rst
