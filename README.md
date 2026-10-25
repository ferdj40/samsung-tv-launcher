# Build your own TV launcher for a Samsung (Tizen)

A custom home screen for a Samsung smart TV: a clock, a photo background, a dock of streaming apps, and rows of movies from your own server. It runs as a normal Tizen web app, so it is plain HTML, CSS and JavaScript.

It cannot replace Samsung's own Home screen. Turn on **Settings > General > Smart Features > Autorun Last App** and the TV opens your launcher at power-on, which is close enough.

Tested on a 2021 Samsung (Tizen 6.0). Use `TV_IP` for your TV's address and `MAC_IP` for your computer's.

## What you need

- A Samsung TV from 2016 or later (Tizen).
- A Mac, Windows or Linux computer on the same network.
- A free Samsung developer account.
- Tizen Studio with the CLI and the TV extensions. About 5 GB.

## 1. Turn on Developer Mode on the TV

1. Open the **Apps** panel on the TV.
2. Type `12345` with the remote's number pad.
3. Switch Developer Mode **On** and type `MAC_IP` as the host PC IP.
4. Unplug the TV for a minute, plug it back in.

`MAC_IP` must be the address your computer uses to reach the TV. If the computer is on Ethernet and Wi-Fi at once, check which interface carries the TV's subnet. A wrong IP here is the most common reason `sdb` cannot connect.

## 2. Make a signing certificate

TVs only run signed apps. In Tizen Studio open **Certificate Manager**, create a Samsung certificate profile, log in with your Samsung account, and add your TV's **DUID**. You get the DUID from the TV once Developer Mode is on:

```
sdb connect TV_IP
sdb -s TV_IP:26101 shell 0 getduid
```

Give the profile a name. The examples below use `MyTV`.

## 3. The smallest app that runs

Make a folder with three files.

`config.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<widget xmlns="http://www.w3.org/ns/widgets" xmlns:tizen="http://tizen.org/ns/widgets"
        id="http://example.local/MyHome" version="1.0.0" viewmodes="maximized">
  <tizen:application id="MyHome01.MyHome" package="MyHome01" required_version="5.0"/>
  <content src="index.html"/>
  <icon src="icon.png"/>
  <name>MyHome</name>
  <access origin="*" subdomains="true"/>
  <tizen:privilege name="http://tizen.org/privilege/internet"/>
  <tizen:privilege name="http://tizen.org/privilege/application.launch"/>
  <tizen:privilege name="http://tizen.org/privilege/application.info"/>
  <tizen:privilege name="http://tizen.org/privilege/tv.window"/>
  <tizen:setting screen-orientation="landscape" context-menu="disable"
                 background-support="enable" hwkey-event="enable"/>
</widget>
```

The package id (`MyHome01`) must be exactly 10 characters. The app id is that package id, a dot, and the app name.

`index.html`: a full-screen `<div id="dock">` and one `<script src="app.js">`.

`icon.png`: 117x117.

Build, sign and install:

```
tizen build-web -out .buildResult -- .
tizen package -t wgt -s MyTV -o ./out -- .buildResult
sdb connect TV_IP
tizen install -n MyHome.wgt -t YOUR_TV_MODEL -- out
```

`YOUR_TV_MODEL` is the name shown by `sdb devices`.

## 4. Launch the TV's real apps

Each installed app has an id. List them:

```
sdb shell 0 applist
```

Netflix, YouTube, Prime Video and the rest each show up with their ids, for example `org.tizen.netflix-app`. Launching one from your app:

```js
tizen.application.launch('org.tizen.netflix-app',
  function () {},
  function (e) { console.log('failed: ' + e.message); });
```

To build the dock without hard-coding every app, ask the TV what is installed:

```js
tizen.application.getAppsInfo(function (list) {
  list.forEach(function (a) { /* a.id, a.name, a.iconPath */ });
});
```

`a.iconPath` points to the app's own icon, and the TV lets your app load most of them. Use it when it loads and fall back to an icon you draw yourself when it does not. Prime Video and Internet failed on the test TV.

Remote buttons arrive as key events. Arrows, OK and Back work as is. The colored buttons need registering:

```js
tizen.tvinputdevice.registerKey('ColorF0Red');
tizen.tvinputdevice.registerKey('ColorF1Green');
```

Long-press on OK does not reach web apps on this TV, so use the red and green buttons for extra actions such as pin, move and remove.

## 5. Rows of movies from your own server

The launcher does not play anything. It reads a list from a small server on your network (any server that returns JSON with a title, a poster URL and an id) and draws the rows. Selecting a movie opens your player app and tells it which movie:

```js
var c = new tizen.ApplicationControl(
  'http://tizen.org/appcontrol/operation/default', null, null, null,
  [new tizen.ApplicationControlData('movie', [movieId])]);
tizen.application.launchAppControl(c, 'YourPlayer01.YourPlayer', ok, fail);
```

The player reads it on start with `tizen.application.getCurrentApplication().getRequestedAppControl()`.

Keep the server address and any token in a separate `config.js` that you do not share.

## 6. The look

- A real photo as background. Rotate a handful every 15 minutes.
- Measure each photo's brightness once, ahead of time, and store it next to the file name. When the photo is bright, switch the text to dark. This is faster and more reliable than reading pixels on the TV.
- Greeting in small capitals, a thin clock, the date, then the focused app's name above a translucent dock.
- The dock holds about six tiles. The movie rows sit below and open with the down arrow.

Photos need a license you can ship with. Wikimedia Commons works; keep a `CREDITS.txt`.

## 7. Things that will bite you

Tizen 6 runs Chromium 76. Skip modern CSS:

- No `inset`. Write `top`, `right`, `bottom`, `left`.
- No `gap` on flexbox. Use margins.
- A modal placed with `inset` ended up off-screen and the app looked frozen.

Other lessons:

- Back should not close the app. Handle the key and stay on the home screen.
- The app is killed when another app opens. With `background-support="enable"` you can watch the running apps with `tizen.application.getAppsContext` and relaunch yourself when the other app closes.
- Test layout in a desktop browser first, then confirm on the TV.
- When something fails only on the TV, send log lines to a tiny HTTP server on your network with `XMLHttpRequest` and read them from your computer. It beats guessing.

## 8. Switching to HDMI inputs (optional, awkward)

`sdb` cannot launch system apps such as the Source menu (error 400), and a web app cannot pick an HDMI input directly. What does work is pressing the remote's HDMI key from outside, over the TV's network remote protocol:

- Run a small Node server on a Raspberry Pi. It opens a WebSocket to `wss://TV_IP:8002` as a named client and sends `KEY_HDMI`.
- The first connection makes the TV ask you to press Allow. The server stores the token the TV returns.
- The launcher calls the Pi: `http://PI_IP:8765/tv?gap=4500&keys=KEY_HDMI,KEY_HDMI`.
- The HDMI key walks a ring: TV Plus, then each input. Read the current input with `tizen.tvwindow.getSource()` and count the presses you need.
- Wait about 4.5 seconds between presses. Faster presses are dropped.
- Guard against double presses of OK, or the count doubles and you land on TV Plus.

This works most of the time. Switching takes 5 to 10 seconds because it passes through TV Plus on the way.

## Reinstalling after a change

```
tizen build-web -out .buildResult -- . \
 && tizen package -t wgt -s MyTV -o ./out -- .buildResult \
 && sdb connect TV_IP \
 && tizen install -n MyHome.wgt -t YOUR_TV_MODEL -- out
```

Bump `version` in `config.xml` for each install so you can tell builds apart.
