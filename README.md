# Electron Audio Loopback

Capture system audio in an Electron application and receive it as a browser `MediaStream`. The package configures Chromium loopback features and bridges Electron's main and renderer processes through IPC.

The npm package name is `electron-audio-loopback`.

## Requirements

- Electron **31.0.1 or later**, as declared by the package's peer dependency.
- Supported platform targets: macOS 12.3+, Windows 10+, and Linux with PulseAudio.
- Any screen or system-audio capture permissions required by the operating system.

Capture behavior depends on the Electron/Chromium version and platform. The declared peer dependency does not guarantee compatibility with every later Electron release.

## Installation

```sh
npm install electron-audio-loopback
```

Your application must also provide Electron.

## Usage

### Initialize the main process

Call `initMain()` once, before the application is ready, so its Chromium feature switches take effect.

```js
// main.js
const { app, BrowserWindow } = require('electron');
const path = require('node:path');
const { initMain } = require('electron-audio-loopback');

initMain();

app.whenReady().then(() => {
  const window = new BrowserWindow({
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
    },
  });

  window.loadFile('index.html');
});
```

### Capture through a preload bridge

Expose the two capture operations from your preload script:

```js
// preload.js
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('loopback', {
  enable: () => ipcRenderer.invoke('enable-loopback-audio'),
  disable: () => ipcRenderer.invoke('disable-loopback-audio'),
});
```

In the renderer, request both audio and video, then stop and remove the video tracks. Use `finally` to clear the temporary display-media handler even if capture fails.

```js
// renderer.js (loaded by index.html)
async function captureSystemAudio() {
  await window.loopback.enable();

  try {
    const stream = await navigator.mediaDevices.getDisplayMedia({
      video: true,
      audio: true,
    });

    for (const track of stream.getVideoTracks()) {
      track.stop();
      stream.removeTrack(track);
    }

    return stream;
  } finally {
    await window.loopback.disable();
  }
}

// Invoke from a capture button's click handler.
document.querySelector('#start').addEventListener('click', async () => {
  try {
    const stream = await captureSystemAudio();
    // Pass stream to MediaRecorder, Web Audio, or RTCPeerConnection.
    // When finished: stream.getTracks().forEach(track => track.stop());
    console.log('Captured audio tracks:', stream.getAudioTracks());
  } catch (error) {
    console.error('System audio capture failed:', error);
  }
});
```

Provide a `<button id="start">Capture system audio</button>` in your page. Stopping the returned tracks ends capture; clearing the IPC handler does not stop an existing stream.

### Use the renderer helper

In a renderer where Electron's `ipcRenderer` and CommonJS `require` are available, the package provides a convenience helper:

```js
const { getLoopbackAudioMediaStream } = require('electron-audio-loopback');

async function capture() {
  const stream = await getLoopbackAudioMediaStream();
  return stream;
}
```

The helper enables loopback capture, requests a display-media stream, stops and removes video tracks by default, clears the display-media handler, and returns the stream. Pass `{ removeVideo: false }` to retain video tracks. The source code cautions that retained video may be unstable.

The helper currently clears the handler only after a successful capture. If `getDisplayMedia()` rejects, clear it through `ipcRenderer.invoke('disable-loopback-audio')` in your error handling, or use the preload bridge pattern above.

## API

### `initMain(options?): void`

Main-process initialization. Preserves existing `enable-features` values while adding the package's loopback feature flags, and registers the capture IPC handlers.

| Option | Type | Default | Behavior |
| --- | --- | --- | --- |
| `sourcesOptions` | Electron `SourcesOptions` | `{ types: ['screen'] }` | Options passed to `desktopCapturer.getSources()`. |
| `onAfterGetSources` | `(sources: DesktopCapturerSource[]) => DesktopCapturerSource[]` | Unset | Synchronously filters or reorders the captured sources. The first returned source is selected; the array must be nonempty. |
| `forceCoreAudioTap` | `boolean` | `false` | Selects the macOS Core Audio Tap feature flag instead of the ScreenCaptureKit override flag. |
| `loopbackWithMute` | `boolean` | `false` | Requests Electron's `loopbackWithMute` audio mode instead of `loopback`. Platform support depends on Electron. |
| `sessionOverride` | Electron `Session` | `session.defaultSession` | Session whose display-media request handler is changed. Use the session associated with the capturing renderer. |

For example, select a particular screen by its source ID:

```js
initMain({
  sourcesOptions: { types: ['screen'] },
  onAfterGetSources: sources => sources.filter(source => source.id === screenId),
});
```

Here, `screenId` must be an application-provided desktop capture source ID.

### `getLoopbackAudioMediaStream(options?): Promise<MediaStream>`

Renderer helper. The sole option is `removeVideo` (`boolean`, default `true`), which controls whether video tracks are stopped and removed before returning the stream.

Runtime exports depend on the Electron process: the main process receives `initMain`, and a renderer receives `getLoopbackAudioMediaStream`.

### IPC channels

| Channel | Behavior |
| --- | --- |
| `enable-loopback-audio` | Installs the session's display-media request handler. Requests select the first configured desktop source and include loopback audio. |
| `disable-loopback-audio` | Clears the session's display-media request handler. |

Both channels use `ipcRenderer.invoke()` and accept no arguments. Avoid registering other handlers on these channel names.

## Capture considerations

- Enabling capture replaces the selected session's display-media handler. Disabling clears it; a previously installed handler is not restored.
- The handler is shared by the session. Coordinate concurrent capture requests and other screen-sharing flows in your application.
- Both `video: true` and `audio: true` are requested, even when you only need audio.
- The default source is the first screen returned by Electron. Use `onAfterGetSources` to change selection.
- Avoid playing the captured audio back through the same output being captured, which can create feedback.
- If capture fails, check operating-system permissions, available desktop sources, the renderer's session, and audio-track availability on your Electron/platform combination.

## Local development

```sh
npm ci
npm run build
```

| Command | Purpose |
| --- | --- |
| `npm run build` | Compile TypeScript and declarations into `dist/`. |
| `npm run dev` | Recompile when source files change. |
| `npm run clean` | Remove `dist/`. |
| `npm run lint` | Run ESLint on `src/`. |
| `npm run lint:fix` | Apply ESLint fixes. |
| `npm test` | Build and launch the interactive Electron example; this is not an automated test suite. |

The bundled example displays an audio player and frequency visualizer. Its main script loads `index.html` using a relative path. To run it with the example directory as the working directory:

```sh
npm run build
cd example
npx electron index.js
```

Source files:

```text
src/
  index.ts       Process-dependent runtime exports
  main.ts        Feature switches and capture IPC handlers
  renderer.ts    MediaStream capture helper
  config.ts      IPC channel names and feature flag configuration
  types.d.ts     Public option interfaces
example/         Interactive Electron demo
```

## License

MIT, as declared in `package.json`. Author: Alec Armbruster.
