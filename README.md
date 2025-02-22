# capacitor-blob-writer
A faster, more stable alternative to @capacitor/filesystem's `Filesystem.writeFile` for writing Blobs to the filesystem.

## Usage
```javascript
/*jslint browser */
import {Directory} from "@capacitor/filesystem";
import {Capacitor} from "@capacitor/core";
import write_blob from "capacitor-blob-writer";

// Firstly, obtain yourself a Blob. This could be a file downloaded from the
// internet, or some binary data generated by your app.

let my_video_blob;

// Secondly, write the Blob to disk. The 'write_blob' function takes an options
// object and returns a Promise. If the Blob is successfully written to disk,
// the Promise is resolved with the absolute path of the newly written file.

write_blob({

// The 'path' option should be a string describing where to write the file. It
// may be specified as an absolute URL (beginning with "file://") or a relative
// path, in which case it is assumed to be relative to the 'directory' option.

    path: "media/videos/funny.mp4",

// The 'directory' option is used to resolve 'path' to a location on the disk.
// It is optional if the 'path' option begins with "file://".

    directory: Directory.Data,

// The 'blob' option must be a Blob, which will be written to the file. The file
// on disk is overwritten, not appended to.

    blob: my_video_blob,

// If the 'recursive' option is 'true', intermediate directories will be created
// as required. It defaults to 'false' if not specified.

    recursive: true,

// If 'write_blob' falls back to its alternative strategy on failure, the
// 'on_fallback' function will be called with the underlying error. This can be
// useful to diagnose slow writes. It is optional.

// See the "Fallback mode" section below for a detailed explanation.

    on_fallback(error) {
        console.error(error);
    }
}).then(async function () {

// Obtain the URI for your platform

    let my_video_uri;
    if (Capacitor.getPlatform() == "web") {
        const { data } = await Filesystem.readFile({ path, directory })
        my_video_uri = URL.createObjectURL(data)
    } else {
        const { uri } = await Filesystem.getUri({ path, directory })
        my_video_uri = Capacitor.convertFileSrc(uri)
    }

// Now you can use it in your video element

    const video_element = document.createElement("video");
    video_element.src = Capacitor.convertFileSrc(my_video_uri);
    document.body.appendChild(video_element);

// Do not forget to revoke the uri when you are done with it!
// This is important to avoid memory leaks!

    video_element.onended = function () {

// The platform check is optional since browsers do not complain
// when an invalid uri is provided.

        if (Capacitor.getPlatform() == "web") {
            URL.revokeObjectURL(my_video_uri);
        }
    }
});
```

## Installation

Different versions of the plugin support different versions of Capacitor:

| Capacitor  | Plugin |
|------------|--------|
| v2         | v0.2   |
| v3         | v1     |

Read the documentation for v0.2 [here](https://github.com/diachedelic/capacitor-blob-writer/tree/0.2.x). See the changelog below for breaking changes.

```sh
npm install capacitor-blob-writer
npx cap update
```

### iOS

Run `Product -> Clean Build Folder` within Xcode if you experience weird runtime errors (#32).

### Android

Configure `AndroidManifest.xml` to [allow cleartext](https://github.com/diachedelic/capacitor-blob-writer/issues/20) communication with the local BlobWriter server.
```xml
<application
    android:usesCleartextTraffic="true"
    ...
```

## How it works
### Web
For the web platform the plugin creates an empty indexedDB entry with `Filesystem.writeFile`. Then it replaces it's size and content with the provided blob, completely avoiding base64 encoding. Therefore, significantly improving the writing speed.

There is a caveat though. If you use `Filesystem.readFile`, you will get a **Blob** as your data back instead of a base64 **string**, since your data has never been encoded in the first place.

### iOS / Android
When the plugin is loaded, an HTTP server is started on a random port, which streams authenticated PUT requests to disk, then moves them into place. The `write_blob` function makes the actual `fetch` call and handles the necessary authentication. Because browsers are highly optimised for network operations, this write does not block the UI.

I had dreamed of having the WebView intercept the PUT request and write the request's body to disk. Incredibly, neither iOS nor Android's webview are capable of correctly reading request bodies, due to [this](https://issuetracker.google.com/issues/36918490) and [this](https://bugs.webkit.org/show_bug.cgi?id=179077). Hence an actual webserver will be required for the forseeable future.

### Fallback mode (iOS / Android)
There are times when `write_blob` inexplicably fails to communicate with the webserver, or the webserver fails to write the file. A fallback mode is provided, which invokes an alternative strategy if an error occurs. In fallback mode, the Blob is split into chunks and serially concatenated on disk using `Filesystem.appendFile`. While slower than `Filesystem.writeFile`, this strategy avoids Base64-encoding the entire Blob at once, making it stable for large Blobs.

## Known limitations & issues
- potential security risk (only as secure as [GCDWebServer](https://github.com/swisspol/GCDWebServer)/[nanohttpd](https://github.com/NanoHttpd/nanohttpd)), and also #12
- no `append` option yet (see #11)
- developers have to handle the URIs for different platforms themselves (see [example](#usage))

## Benchmarks
I have compared the performance & stability of `Filesystem.writeFile` with `write_blob` on my devices, see `demo/src/index.ts` for more details.

### Android (Samsung A5)

| Size          | Filesystem       | BlobWriter          |
|---------------|------------------|---------------------|
| 1 kilobyte    | 18ms             | 89ms                |
| 1 megabyte    | 1009ms           | 87ms                |
| 8 megabytes   | 10.6s            | 0.4s                |
| 32 megabytes  | Out of memory[1] | 1.1s                |
| 256 megabytes |                  | 17.5s               |
| 512 megabytes |                  | Quota exceeded[2]   |

- [1] Crash `java.lang.OutOfMemoryError`
- [2] File cannot be moved into the app's sandbox, I assume because the app's disk quota is exceeded

### iOS (iPhone 6)

| Size          | Filesystem       | BlobWriter          |
|---------------|------------------|---------------------|
| 1 kilobyte    | 6ms              | 16ms                |
| 1 megabyte    | 439ms            | 26ms                |
| 8 megabytes   | 3.7s             | 0.2s                |
| 32 megabytes  | Out of memory[1] | 0.7s                |
| 128 megabytes |                  | 3.1s                |
| 512 megabytes |                  | WebKit error[2]     |

- [1] Crashes the WKWebView, which immediately reloads the page
- [2] `Failed to load resource: WebKit encountered an internal error`

### Google Chrome (Desktop FX-4350)
The plugin puts your blob directly[1] into the indexedDB storage.

| Size          | Filesystem       | BlobWriter          |
|---------------|------------------|---------------------|
| 1 kilobyte    | 4ms              | 9ms                 |
| 1 megabyte    | 180ms            | 16ms                |
| 8 megabytes   | 1.5s             | 43ms                |
| 32 megabytes  | 5.2s             | 141ms               |
| 64 megabytes  | 10.5s            | 0.2s                |
| 512 megabytes | Error[2]         | 1.1s                |

- [1] Data returned from `Filesystem.readFile` is your pure blob (not base64 encoded)
- [2] `DOMException: The serialized keys and/or value are too large`

## Changelog

### v1.0.0
- BREAKING: `write_blob` is now the default export of the capacitor-blob-writer package.
- BREAKING: `write_blob` returns a string, not an object.
- BREAKING: The `data` option has been renamed `blob`.
- BREAKING: The `fallback` option has been removed. Now, fallback mode can not be turned off. However you can still detect when fallback mode has been triggered by supplying an `on_fallback` function in the options.
- BREAKING: Support for Capacitor v2, and hence iOS v11, has been dropped.
- Adds support for Capacitor v3.
