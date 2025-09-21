# Concept Questions

1. The context is used to create separate scopes for uniqueness within the NonceGeneration concept. In the URL shortening app, the context would be the domain, such as "tinyurl.com" or "bit.ly". This allows the same suffix to be reused across different domains while still maintaining uniqueness within each domain. For example, "tinyurl.com/abc123" and "bit.ly/abc123" can both exist since they are different contexts, but within the domain "tinyurl.com", each generated suffix needs to be unique.

2. The NonceGeneration stores sets of used strings to ensure uniqueness and track what is already being used. For instance, the simple implementation prevents us from being able to distinguish what strings/suffixes have already been used, as well as making it impossible to modify existing suffixes in the future. As far as how the counter and the set implementation relates to each other, you can simply count the size of the set to achieve the same functionality of the counter approach.

3. One advantage of this idea is that it's quite memorable and easy to understand. For example, if I was running out of time to copy the link down, remembering the word allows me to finish the shortlink, as opposed to a string composed of random chars/numbers. A disadvantage is that you severely limit the amount of generations you can create by limiting to "common dictionary words", and it will not scale. We could modify NonceGeneration such that the state is a set of Contexts with a used set of DictWords, where DictWord is a separate type that defines a "common dictionary word".

# Synchronization Questions

1. In the first sync, it is likely that the action is purely for generating a short url, not necessarily creating a binding or mapping. In the second sync, you are more explicitly registering a short link that will be binded to a target URL, so it must be provided.

2. The omitting names convention can only be used when the variable name exactly matches the parameter name. In cases like `NonceGeneration.generate (context: shortUrlBase)` or `UrlShortening.register (shortUrlSuffix: nonce)`, explicit naming is required because the variable names (`shortUrlBase`, `nonce`) differ from the parameter names (`context`, `shortUrlSuffix`).

3. Because the third sync is something handled on the server, and not the client. It is more of a "side-effect" upon handling the request of a user.

4. No baseURL is needed since we know it will always be "bit.ly".

sync generate

- when Request.shortenUrl (targetUrl)
- then NonceGeneration.generate ()

sync register

- when Request.shortenUrl (targetUrl)
  NonceGeneration.generate (): (nonce)
- then UrlShortening.register (shortUrlSuffix: nonce, targetUrl)

5. A sync to handle resource expiration:

```
sync expireUrl
when ExpiringResource.expireResource(): (resource: Resource)
then UrlShortening.delete(shortUrl: resource: Resource)
```

This sync will triger when the ExpiringResource concept automatically expires a resource (the short URL), and then deletes the corresponding URL mapping from the UrlShortening concept.

# Extending the Design

concept ClickAnalytics

purpose

- track usage statistics for shortened URLs

principle

- after recording clicks for a URL, getClickCount returns the total number

state

- a set of Analytics with
  a shortUrl String
  a clickCount Number

actions

- recordClick (shortUrl: String)

  - effect increments the click count for shortUrl (creates entry with count 1 if none exists)

- getClickCount (shortUrl: String): (count: Number)

  - requires analytics exists for shortUrl

  - effect returns the click count for the given short URL

- resetCount (shortUrl: String)

  - requires analytics exists for shortUrl

  - effect sets click count for shortUrl to 0

concept UserOwnership \[User\]

purpose

- associate resources with users

principle

- after assigning ownership of a resource to a user, only that user can access/change it

state

- a set of Ownerships with
- a resource String
- a owner User

actions

- assignOwner (resource: String, owner: User)

  - requires no ownership exists for resource

  - effect creates ownership record associating resource with owner

- getOwner (resource: String): (owner: User)

  - requires ownership exists for resource

  - effect returns the owner of the given resource

- verifyOwner (resource: String, user: User): (isOwner: Boolean)

  - effect returns true if user owns resource, false otherwise

- transferOwnership (resource: String, newOwner: User)

  - requires ownership exists for resource

  - effect changes owner of resource to newOwner

sync assignOwnership

- when UrlShortening.register (): (shortUrl)
- then UserOwnership.assignOwner (resource: shortUrl, owner: currentUser)

sync trackClick

- when UrlShortening.lookup (shortUrl)
- then ClickAnalytics.recordClick (shortUrl)

sync viewAnalytics

- when Request.getAnalytics (shortUrl, user)
  UserOwnership.verifyOwner (resource: shortUrl, user): (isOwner)
  - requires isOwner == true
- then ClickAnalytics.getClickCount (shortUrl): (count)

1. Modify NonceGeneration to accept optional user-provided nonce, add validation sync to check availability before registration.

2. Undesirable, limits scale and also difficult to check for type safety.

3. Extend ClickAnalytics state to include targetUrl field, modify recordClick to accept both shortUrl and targetUrl.

4. Modify NonceGeneration to use more "random" generation or longer suffixes instead of shorter, sequential/predictable patterns.

5. Undesirable, as by definition, analytics should not be public but should be viewable only by the user who registered the shortening.
