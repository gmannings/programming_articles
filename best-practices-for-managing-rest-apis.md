# Best Practices for Managing and Deprecating RESTful API Endpoints

Between significant versions of REST APIs there may be changes to the data structure that necessitate changing
the API interface. Managing this change is often worrisome for APIs that have external users as communicating the
changes can be difficult. With this page I work through some mechanisms for RESTful API interface change managemenent
and deprecating endpoints.

## API versioning

There are a number of approaches to versioning an API which can be used:

* Version as part of URL: `/api/v1/my/resource` would be version 1 and `/api/v2/my/resource` would be version 2
  * **Pros**: This is straightforward to implement, and easily understandable
  * **Cons**: If the level of change on the API is high, then this can cause problems for clients to update. The
  version information becomes part of the resource URL, which if used as key or identifier in client systems, will
  potentially cause more breaking changes when a client upgrades.
* Query parameter versioning: `/api/my/resource?version=1`
  * **Pros**: Straightforward to implement in routing layer. If the parameter is not supplied then the latest version
  can be supplied, making it a choice for a developer to maintain usage of older API versions.
  * **Cons**: Can cause caching complications. The version information is abstracted away from the resource
* Header versioning through the supply of `X-APIName-Version: 1` which in a similar way to parameter matching can default
  to the latest version.
  * **Pros**: API version information is not part of the resource URL request, which allows for cleaner URLs that remain
  unchanged across versions if the resource remains the same.
  * **Cons**: Reduces transparency as versioning is not visible in the URL and can make debugging harder.

There are other methods such as API version in the hostname or changing version based off content negotiation through
the `Accept` header, but the most common I've seen are described above.

My personal preference is for headers to be used to switch between API versions as I prefer my URLs to remain consistent
between versions (even if the underlying representation changes) so I will use this as a base for the rest of the 
examples. I also prefer pushing to the latest version of the API if no header is supplied, however this does mean
that the potential for breaking changes are higher, at the expense of reducing client lag in using the latest
versions.

### Semantic versioning

For simplicity, I will not detail the differences in version numbers and how to manage these. Instead  I will concentrate
on an API that maintains semantic versioning:

```
{MAJOR}.{MINOR}.{PATCH}
```

Defining each of these values as:

* MAJOR version when you make incompatible API changes,
* MINOR version when you add functionality in a backward-compatible manner, and
* PATCH version when you make backward-compatible bug fixes.

For changes that do not affect the API interface or representation data, these are generally MINOR or PATCH changes to 
the API. Where changes affect the interface or _change_ the representation, these are likely to be MAJOR changes as they
would break any existing clients.

## Managing change where the default API is the latest

If the API is changing frequently it is clear to see that the MAJOR version of the API may increase substantially
if the code is continually integrated (CI). If not using CI then releases may be less frequent, however the same 
problems occur no matter what the integration and release strategy - how to inform users of the change and ensure
that systems are not broken.

Assuming there are 100 consumers currently using the API, and only a handful are specifying a version of the API that
they wish to consume, there is a risk that introducing a breaking change will cause significant problems, so we have
to manage change carefully.

### Example of change

For the change example we have an API that requires a change in ID management, to move away from numeric IDs to UUIDs.
An example of this change is:

`/user/123456` to `user/1a2b3c4d-5e6f-7g8h-9i0j-123456789abc`

This is a MAJOR change so the semantic version of the API shall be increased from version `1.3.6` to `2.0.0`.

The API implements header versioning, and if the header is not defined, the response will automatically return the 
latest version.

### How to handle

As we don't want to break our consumer clients immediately on release, and don't want to hold the change to the API
from being integrated, we need to carefully manage the process.

1. Still provide the old `/user/{numericId}` endpoint as well as the new `/user/{UUID}` endpoint for a fixed amount
   of time (in this example, 6 months when the next MAJOR change is anticipated).
2. Change the old endpoint into a facade, where the new endpoint controller is called after retrieving the user's new
   UUID, therefore removing most of the legacy code. Annotate this endpoint with some `deprecated` metadata, and if
   possible a version of the API where it will be removed (in this case `3.X.X`).
3. Add a `Link` header that provides the updated resource URL for the user: `Link: </user/{UUID}>; rel="alternate"`
4. Optionally add the non-standard, but well understood, `Deprecation` header: `Deprecation: date="2024-07-01"`
5. Update any Swagger documentation with the deprecation notice
6. Communicate to clients of the impending change.

At this point the change has effectively been managed and clients that still require the old endpoint style would have
to specify version `2.0.0` of the API once version `3.X.X` is released. This then allows the old API to be supported
as per business SLAs.

### Hidden benefits of this approach

If the API has not previously been versioned then a change like this can often be viewed as 'impossible', and leads
to a new API being developed on a new domain so that users can be slowly transitioned. By implementing header versioning
and communicating the change to consumers, this provides the mechanism for future significant change without duplicating
API services or breaking existing implementations.

## Managing change where the API version must be specified in requests

Where an API forces a client to provide the version header, this eradicates the problem of breaking existing implementations.
It however causes a new problem - clients using old versions of the API and not updating to latest versions. This change
management can be more difficult as systems that lag with few updates don't generally look at their HTTP response headers
to make note of `Deprecation` headers. When old versions of the API are to be discontinued, it can be difficult to
manage the change.

### How to handle

1. Inform users via documentation and communication of the retirement date of API versions.
2. Add `Deprecation` headers to formalise the changing endpoints AND/OR use the non-standard `Sunset` header to specify
   when this API version will no longer be supported.
3. Monitor logs for consumers still using the unsupported version and inform them of the impending change

## Conclusion

Managing change on REST APIs is not necessarily difficult or complex (depending on the change), however implementing
change mechanisms, not breaking clients, and communicating the changes to public users can be. Having a clear plan
on how these changes will be managed will reduce risk and uncertainty when inevitable changes are made. 
