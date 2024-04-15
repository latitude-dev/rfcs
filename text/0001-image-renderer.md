- Start Date: 2024-040-15
- RFC PR: (leave this empty)
- Latitude Issue: https://github.com/latitude-dev/latitude/issues/296

# Summary

At latitude we want to allow users to render the views (data apps) they do as an
image. This is helpful for embeding those images in emails, reports, Slack
channels, etc.

# Basic example

The most basic use of this feature would be to send a new url param called
`__format`. When format is `image` the view is rendered as an image.

Example:

```bash
curl https://your-app.latitude.page/some/view/path?__format=image
```

# Motivation

The motivation is to make easier for users to render reports as images to use in
different mediums. Rendering a webpage as an image is not straightforward and we
think we can help with that.

# Detailed design

The main idea is that rendering a Latitude view as an image should as simple as
posible. We will use the `__format=image` in a views's url to trigger the image
rendering.

Under the hood building the image will be done by a headless browser. We will use [Playwright](https://playwright.dev/)
to do this. The image will be rendered in the same size as the view is rendered in the browser.

Optionally users can specify the size of the image by using the `__width` or `__height` url params. (not sure about this)

### Arquitecture
Current arquitecture has the Latitude app under `apps/server`. The app is a
SvelteKit app. We will add a new app under `apps/renderer` that will handle the
rendering of the images.

Rendering a headless browser is a heavy operation. We will need to cache the images. We will set a disk cache for the images.
Eventually can add a CDN to cache the images in S3 or Cloudflare,...

Cache keys are the URL of the image hashed.
Example of hashing the URL in Node.js:

```javascript
import { createHash } from 'crypto'
// Remove __format=image from the URL before hashing
const hash = createHash('sha256')
const viewKey = createHash('sha256')
    .update('https://your-app.latitude.page/some/view/path?some=params&and=more')
    .digest('hex')
```

With this hash we generate the image file name. The image will be stored in a
place that's not accessible from outside the app. (Let's talk about privacy/security
concerns)

The logic is like this:

1. Some user or user's server request a Latitude view as an image
2. `apps/view` receives the request and sends a message to `apps/renderer` to render the image because `__format=image` is present in the URL
3. `apps/renderer` receives the message and hash the URL from that view and can happen two things:
    3.1. The image is already cached. In this case the image is served from the disk/cdn cache.
    3.2. The image is not cached. In this case the image is rendered by Playwright and stored in the cache
4. `apps/renderer` sends the image path to `apps/view` and the user can download the image

### Pieces
- Cache system of the generated images. We do this on disk for now
- [Playwright](https://playwright.dev/) to render the images
- [Bull/Redis](https://optimalbits.github.io/bull/) to queue the image rendering

### Sync process

From a user's perspective the image rendering is a sync process. The user sends a request and we return image from disk or produce the image and return it.

                                                                                                                                             │                                       │
                                                                                                                                             │                                       │
                                                                  ┌───────────────────────────────┐                                          │                                       │
                                                                  │                               │                                          │                                       │
                                                                  │                               │          enqueue render image            │                                       │
      ┌────────────────────────────────────┐                      │                               ├────────────────────────────────────────► │                                       │
      │                                    │  ?__format=image     │                               │                                          │                                       │
      │                                    │                      │                               │                                          │                                       │
      │                                    │                      │                               │                                          │            renderer service           │
      │       user or user's server        ├─────────────────────►│          apps/server          │                                          │                                       │
      │                                    │                      │                               │                                          │                                       │
      │                                    │                      │                               │                                          │                                       │
      │                                    │                      │                               │          render image finish event       │                                       │
      │                                    │                      │                               │◄─────────────────────────────────────────┤                                       │
      └────────────────────────────────────┘                      │                               │                                          │                                       │
                                                                  │                               │                                          │                                       │
                                                                  └───────────────────────────────┘                                          │                                       │
                                                                                                                                             └───────────────────────────────────────┘

### Pseudo code

```javascript
  // Bull/redis queue system that's called from `apps/server`
  const job = await renderImageQueue.add({ data: req.body.data });

  // Job can return a promise so the request feels sync for the user
  job.finished().then((result) => {
    res.json(result)
  }).catch((err) => {
    res.status(500).json(err)
  })
```


# Drawbacks

The main drawback is that we are adding a new service to the Latitude app. This
complicates the setup of the app. We need to setup a new app, a new queue
system. But I think we're solving a real problem here. Also this is opt-in so
if use1rs don't want to use it they don't need to setup the renderer service.

# Alternatives

I considered a request image and then send built image to the user's server in a
callback. This way requesting images is faster but the process is more
complicated to setup for the users. So we'll go w1ith the sync process.

# Adoption strategy

If we implement this proposal, how will existing Latitude developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

Is a new feature that wasn't before so users will need to know about it. We will
need to setup some documentation and applied example.

# Unresolved questions

A classic. `Cache invalidation`. How long should be valid the image for a view?
Should we 1allow users to invalidate the cache? How? Should we use a CDN for the images?
