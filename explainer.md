# video.requestAnimationFrame() Explainer

# Introduction
Today [`<video>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement) elements have no means by which to signal when a video frame has been presented for composition nor any means to provide metadata about that frame.

We propose a new HTMLVideoElement.requestAnimationFrame() method with an associated VideoFrameRequestCallback to allow web authors to identify when and which frame has been presented for composition.


# Use cases

Today sites using [WebGL](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API) or [`<canvas>`](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API) with a [`<video>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement) element rely on a set of heuristics to determine if the first frame has been presented ([video.currentTime > 0](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/currentTime), video [canplaythrough](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/canplaythrough_event) event, etc). These heuristics vary across browsers. For subsequent frames sites must blindly call [Canvas.drawImage()](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/drawImage) or [GLContext.texImage2D()](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/texImage2D) for access to the pixel data. Our proposed API would allow reliable access to the presented video frame.

In an era of [Media Source Extensions](https://developer.mozilla.org/en-US/docs/Web/API/Media_Source_Extensions_API) based [adaptive video playback](https://en.wikipedia.org/wiki/Adaptive_bitrate_streaming) (e.g., [YouTube](https://www.youtube.com/), [Netflix](https://www.netflix.com/), etc) and boutique [WebRTC](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API) streaming solutions (e.g., [Rainway](https://rainway.com/), [Stadia](https://store.google.com/us/magazine/stadia)), the inherent raciness of access operations and the lack of metadata exposed about the frames limits quality and automated analysis. Our proposed API would allow reliable access to metadata about which frame has been presented when for correlation with other page level events (user input, etc).

Additionally, our proposal will enable will enable a host of frame-accurate [web-platform-tests](https://github.com/web-platform-tests/wpt) which can be shared across browsers that were heretofore impossible or otherwise flaky.


# Proposed API

```Javascript
dictionary VideoFrameMetadata {
    // The presentation timestamp in seconds of the frame presented.
    required double presentationTimestamp;

    // The time at which the user agent submitted the frame for composition.
    required DOMHighResTimeStamp presentationTime;

    // The time at which the user agent expects the frame to be visible.
    required DOMHighResTimeStamp expectedPresentationTime;

    // The width and height of the presented video frame.
    required unsigned long width;
    required unsigned long height;

    // Potentially other useful properties? Some surfaced from WebRTC?
};

callback VideoFrameRequestCallback = void(VideoFrameMetadata);

partial interface HTMLVideoElement {
    void requestAnimationFrame(VideoFrameRequestCallback callback);
};
```


# Example

```Javascript
  let video = document.createElement('video');
  let canvas = document.createElement('canvas');
  let canvasContext = canvas.getContext('2d');

  let frameInfoCallback = metadata => {
    console.log(
      `Presented frame ${metadata.presentationTimestamp}s ` +
      `(${metadata.width}x${metadata.height}) at ` +
      `${metadata.presentationTime}ms for display at ` +
      `${expectedPresentationTime}ms`);

    canvasContext.drawImage(video, 0, 0, metadata.width, metadata.height);
    video.requestAnimationFrame(frameInfoCallback);
  };

  video.requestAnimationFrame(frameInfoCallback);
  video.src = 'foo.mp4';
```

Output:
```Text
Presented frame 0s (1280x720) at 1000ms for display at 1016ms.
```


# Implementation Details
* When texImage2D() or drawImage() is called during an active VideoFrameRequestCallback, implementations should ensure that the video frame used is the one matching the active callback.
* Just like window.requestAnimationFrame(), callbacks are one-shot. video.requestAnimationFrame() must be called again for the next frame.


# Open Questions / Notes
* `requestAnimationFrame` is a bit of a misnomer and the proposed API diverges from [window.requestAnimationFrame()](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame) -- we should either rename or not diverge too much (e.g., allow multiple video.requestAnimationFrame() calls that are cancellable and identifiable by some integer id).
* Which properties would be useful to surface from WebRTC?
* Should we also surface playback quality statistics? E.g., time taken to decode, etc.
* We should hang the VideoFrameMetadata structure off of [WebGLTexture](https://developer.mozilla.org/en-US/docs/Web/API/WebGLTexture), so that for DOM-less WebGL usage where texImage2D is the only compositor we avoid the need for requestAnimationFrame() to get frame metadata.
* The API as proposed will miss some frames when compositing is done off the main thread if the next video.requestAnimationFrame() call does not happen in time. To rectify this we would need to make callbacks repeating and provide a cancellation mechanism.
