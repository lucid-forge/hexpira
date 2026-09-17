# Browser reader

This directory is reserved for rendering, image detection, perspective
correction, and webcam integration. Browser processing is intentionally kept
outside `src/core` so the wire codec can be ported without DOM or Canvas APIs.

The current working implementation is in `reference/web/index.html`.


