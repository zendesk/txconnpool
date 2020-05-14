[![codecov](https://codecov.io/gh/zendesk/txconnpool/branch/master/graph/badge.svg)](https://codecov.io/gh/zendesk/txconnpool)

# txconnpool

## Description

txconnpool is a generalized connection pooling library for Twisted.

Assume that we've got a web application, which performs some expensive 
computations, and then caches them in a [memcached](http://memcached.org/) server.  The simple way to
achieve this in Twisted is to create a [ClientCreator](http://twistedmatrix.com/documents/current/api/twisted.internet.protocol.ClientCreator.html) 
for the [MemCacheProtocol](http://twistedmatrix.com/documents/current/api/twisted.protocols.memcache.MemCacheProtocol.html) 
and whenever we need to communicate with the server, we can simply use that.

This works for low volumes of queries, but let's say that now we start hitting memcached a lot--several times per web request, of which we are receiving many per second.  Very quickly, the connection overhead can become a problem.

Instead of creating a new connection for every query, it would be much better to maintain a pool of open connections, and simply reuse those open connections; queuing up any queries if all of the connections are in use.  With txconnpool, setting this up can be quite easy.

## Owners

While anyone can contribute to the repository, the primary owner of this repository is **[Pikachu](https://cerebro.zende.sk/teams/pikachu)**, located in Singapore.

| Channel  | Contact                                                           |
|----------|-------------------------------------------------------------------|
| Slack    | [Pikachu](https://zendesk.slack.com/archives/ask-pikachu)         |
| Email    | pikachu@zendesk.com                                               |
| GitHub   | [@zendesk/pikachu](https://github.com/orgs/zendesk/teams/pikachu) |

We are working in **Singapore Time** (SGT) (UTC +8).

## Table of Contents

* [Example Usage](#example-usage)
* [Contributing](#contributing)

## Example Usage

First we need to create a few classes of boilerplate, to transform a
`MemCacheProtocol` into a `PooledMemcachedProtocol`, and then create a pool:

```
from twisted.protocols.memcache import MemCacheProtocol

from txconnpool.pool import PooledClientFactory, Pool

class PooledMemCacheProtocol(MemCacheProtocol):
    """
    A MemCacheProtocol that will notify a connectionPool that it is ready
    to accept requests.
    """
    factory = None

    def connectionMade(self):
        """
        Notify our factory that we're ready to accept connections.
        """
        MemCacheProtocol.connectionMade(self)

        self.factory.connectionPool.clientFree(self)

        if self.factory.deferred is not None:
            self.factory.deferred.callback(self)
            self.factory.deferred = None

class MemCacheClientFactory(PooledClientFactory):
    protocol = PooledMemCacheProtocol

class MemCachePool(Pool):
    clientFactory = MemCacheClientFactory

    def get(self, *args, **kwargs):
        return self.performRequest('get', *args, **kwargs)

    def set(self, *args, **kwargs):
        return self.performRequest('set', *args, **kwargs)

    def delete(self, *args, **kwargs):
        return self.performRequest('delete', *args, **kwargs)

    def add(self, *args, **kwargs):
        return self.performRequest('add', *args, **kwargs)

```

Now, with this having been created, we can go ahead and use it:
```
from twisted.internet.address import IPv4Address
    
addr = IPv4Address('TCP', '127.0.0.1', 11211)
mc_pool = MemCachePool(addr, maxClients=20)

d = mc_pool.get('cached-data')

def gotCachedData(data):
    flags, value = data
    if value:
        print('Yay, we got a cache hit')
    else:
        print('Boo, it was a cache miss')

d.addCallback(gotCachedData)
```

## Contributing

- [Useful guide to good commit messages](http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)
- Commit messages should _not_ contain issue key. Put them in the PR instead. There is nothing wrong with having the issue IDs in the commits, but putting them only in the PR will save some characters per commit.
- Each commit message should be a representative of what that commit contributes to the repo, not something unhelpful like "Add tests", or "Cleanup" or "JIRA-123FU". By looking at the message one should have a reasonable understanding of the purpose behind the commit.
- You need 2 :+1:s and a green Travis build to merge PRs.
- For major changes, get someone from Pikachu to review the PR. (Ping [#ask-pikachu](https://zendesk.slack.com/archives/ask-pikachu))