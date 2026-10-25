# Build your own home screen for a Samsung TV

I wrote a custom launcher for my Samsung TV: a clock, a photo background, a dock of streaming apps, and rows of movies from my own server. People keep asking how it works, so here is the whole process.

It is a normal Tizen web app. HTML, CSS, JavaScript, no framework.

One limit up front: an app cannot replace Samsung's Home screen. Turn on **Settings > General > Smart Features > Autorun Last App** and the TV opens your launcher at power-on. In practice that is enough.

Tested on a 2021 Samsung (Tizen 6.0). Placeholders used below: `TV_IP` is your TV's address, `MAC_IP` is your computer's, `PI_IP` is a Raspberry Pi if you use one.

## What you need

- A Samsung TV from 2016 or later
- A computer on the same network
- A free Samsung developer account
- [Tizen Studio](https://developer.tizen.org/development/tizen-studio/download) with the CLI and the TV extensions (about 5 GB)

## 1. Turn on Developer Mode

1. Open the **Apps** panel on the TV.
2. Type `12345` on the remote's number pad.
3. Switch Developer Mode on and enter `MAC_IP` as the host PC IP.
4. Unplug the TV for a minute, then plug it back in.

`MAC_IP` has to be the address your computer uses to reach the TV. If your computer is on Ethernet and Wi-Fi at once, use the one on the TV's network. A wrong address here is the usual reason `sdb` cannot connect.

## 2. Create a signing certificate

TVs only run signed apps. In Tizen Studio open **Certificate Manager**, create a Samsung certificate profile, sign in with your Samsung account, and add your TV's DUID. Get the DUID with:

```sh
sdb connect TV_IP
sdb shell 0 getduid
```

Name the profile something short. The commands below use `MyTV`.

## 3. A launcher in three files

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

The package id (`MyHome01`) must be exactly 10 characters. The app id is the package id, a dot, and the app name.

`index.html`:

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=1920,height=1080">
  <title>MyHome</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div id="clock"></div>
  <div id="dock"></div>
  <script src="app.js"></script>
</body>
</html>
```

`style.css`:

```css
body { margin: 0; width: 1920px; height: 1080px; background: #101820; color: #fff;
       font-family: sans-serif; overflow: hidden; }
#clock { position: absolute; top: 80px; left: 120px; font-size: 120px; font-weight: 200; }
#dock { position: absolute; bottom: 160px; left: 120px; display: flex; }
.tile { width: 220px; height: 220px; margin-right: 24px; border-radius: 28px;
        background: rgba(255,255,255,.12); font-size: 32px;
        display: flex; align-items: center; justify-content: center; }
.tile.focus { background: rgba(255,255,255,.35); transform: scale(1.12); }
```

`app.js`:

```js
var TILES = [
  { name: 'Netflix',  id: 'org.tizen.netflix-app' },
  { name: 'YouTube',  id: '9Ur5IzDKqV.TizenYouTube' },
  { name: 'Prime',    id: 'org.tizen.primevideo' },
  { name: 'Disney+',  id: 'MCmYXNxgcu.DisneyPlus' },
  { name: 'Plex',     id: 'kIciSQlYEM.plex' },
  { name: 'Spotify',  id: 'rJeHak5zRg.Spotify' }
];
var focus = 0, dock = document.getElementById('dock');

function draw() {
  dock.innerHTML = '';
  TILES.forEach(function (t, i) {
    var el = document.createElement('div');
    el.className = 'tile' + (i === focus ? ' focus' : '');
    el.textContent = t.name;
    dock.appendChild(el);
  });
}

function launch(t) {
  tizen.application.launch(t.id, function () {}, function (e) {
    console.log('could not open ' + t.name + ': ' + e.message);
  });
}

document.addEventListener('keydown', function (e) {
  var k = e.keyCode;
  if (k === 37 && focus > 0) focus--;
  else if (k === 39 && focus < TILES.length - 1) focus++;
  else if (k === 13) launch(TILES[focus]);
  else return;
  e.preventDefault();
  draw();
});

function tick() {
  var d = new Date(), h = d.getHours(), m = d.getMinutes();
  document.getElementById('clock').textContent = (h % 12 || 12) + ':' + (m < 10 ? '0' : '') + m;
}
tick(); setInterval(tick, 20000); draw();
```

Add a 117x117 `icon.png`, then build, sign and install:

```sh
tizen build-web -out .buildResult -- .
tizen package -t wgt -s MyTV -o ./out -- .buildResult
sdb connect TV_IP
tizen install -n MyHome.wgt -t YOUR_TV_MODEL -- out
```

`YOUR_TV_MODEL` is the name `sdb devices` prints. Bump `version` in `config.xml` on every install so you can tell builds apart.

## 4. Find the real app ids

The ids in the example are the ones on my TV. Yours may differ, so list them:

```sh
sdb shell 0 applist
```

Or let the TV tell the app at runtime, so nothing is hard-coded:

```js
tizen.application.getAppsInfo(function (list) {
  list.forEach(function (a) { /* a.id, a.name, a.iconPath */ });
});
```

`iconPath` is the app's own icon. The TV lets your app load most of them, so use it when it loads and draw your own fallback when it does not. On my TV 17 of 19 loaded; Prime Video and Internet failed.

## 5. Remote buttons

Arrows, OK and Back arrive as key events. The colored buttons have to be registered first:

```js
tizen.tvinputdevice.registerKey('ColorF0Red');
tizen.tvinputdevice.registerKey('ColorF1Green');
```

A long press on OK never reached my app, so extra actions (pin, move, remove) live on the red and green buttons.

## 6. Rows of movies from your own server

The launcher plays nothing. It reads a JSON list (title, poster URL, id) from a small server on your network and draws the rows. Selecting a movie opens your player app and tells it which one:

```js
var c = new tizen.ApplicationControl(
  'http://tizen.org/appcontrol/operation/default', null, null, null,
  [new tizen.ApplicationControlData('movie', [movieId])]);
tizen.application.launchAppControl(c, 'Player001.Player', ok, fail);
```

The player picks it up at start with `tizen.application.getCurrentApplication().getRequestedAppControl()`. Keep the server address and any token in a separate `config.js` that stays out of git.

## 7. The look

- A real photo as background, with a few rotating every 15 minutes.
- Measure each photo's brightness once, ahead of time, and store it beside the file name. When the photo is bright, switch the text to dark. That is more reliable than reading pixels on the TV.
- A small greeting, a thin clock, the date, then the focused app's name above a translucent dock of about six tiles.
- Movie rows sit below the dock and open with the down arrow.

Use photos you are allowed to ship. Wikimedia Commons works; keep a `CREDITS.txt`.

## 8. What will bite you

Tizen 6 runs Chromium 76, so modern CSS is out:

- No `inset`. Write `top`, `right`, `bottom`, `left`. A modal I positioned with `inset` landed off-screen and the app looked frozen.
- No `gap` on flexbox. Use margins.

More lessons:

- Handle Back yourself. Mine returns to the first row and never closes the app.
- With `background-support="enable"` you can poll `tizen.application.getAppsContext` and relaunch your launcher when the app you opened closes.
- Build the layout in a desktop browser first, then confirm on the TV.
- When something breaks only on the TV, post log lines to a tiny HTTP server on your network with `XMLHttpRequest` and read them from your computer.

## 9. Switching HDMI inputs (optional, awkward)

`sdb` cannot launch system apps like the Source menu (it returns error 400), and a web app cannot choose an HDMI input directly. What does work is sending the remote's HDMI key from outside, over the TV's network remote protocol:

1. Run a small Node server on a Raspberry Pi. It opens a WebSocket to `wss://TV_IP:8002` as a named client and sends `KEY_HDMI`.
2. The first connection makes the TV ask you to press Allow. The server saves the token the TV returns.
3. The launcher calls the Pi: `http://PI_IP:8765/tv?gap=4500&keys=KEY_HDMI,KEY_HDMI`.
4. The HDMI key walks a ring: TV Plus, then each input. Read the current input with `tizen.tvwindow.getSource()` and count the presses you need.
5. Wait about 4.5 seconds between presses. Faster presses get dropped.
6. Ignore a second OK while a switch is running, or the count doubles and you land on TV Plus.

It works most of the time. A switch takes 5 to 10 seconds because it passes through TV Plus.

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| `sdb connect` fails | Wrong host IP in Developer Mode. Fix it, unplug the TV, wait a few minutes. |
| Install says certificate error | The TV's DUID is not in your certificate profile. |
| App installs but the screen is blank | Check for `inset` or `gap`; test in a browser first. |
| An app will not launch | Wrong id. Compare with `sdb shell 0 applist`. |

## Sideloading FAQ

**Can I sideload apps on a Samsung TV?**
Yes. Turn on Developer Mode, make a signing certificate, and install a Tizen `.wgt` package with `sdb` and the `tizen` CLI. That is the same flow as step 3.

**Can I install an APK?**
No. Samsung TVs run Tizen, not Android. Only `.wgt` web apps install.

**Can I install a `.wgt` someone else built?**
Not as it comes. The TV only accepts packages signed with a certificate that includes your TV's DUID. Unzip the `.wgt`, then repackage and sign it with your own profile:

```sh
tizen package -t wgt -s MyTV -o ./out -- ./unzipped-app
```

If the app's `config.xml` has an id that clashes with an app already on the TV, change the package id first.

**Do I need a Samsung account?**
Yes, a free one. Certificate Manager uses it to issue your certificate.

**Does it work on every Samsung TV?**
Only Tizen models from about 2016 on. Every TV needs its own DUID in the certificate, so a certificate made for one TV will not install on another until you add that TV.

**Does the install last?**
Samsung developer certificates expire. Check the validity dates in Certificate Manager, and re-sign and reinstall before the date passes.

**Why does `sdb connect` fail?**
Almost always the host IP in Developer Mode. It must be the address your computer uses to reach the TV. After changing it, unplug the TV and wait a few minutes; it can show as offline before it connects.

**Why does a sideloaded app say "launch service failed"?**
Some apps ship with a background service that does not start on every TV. I hit this with a standalone YouTube-client build. Another build of the same app, which needed the host IP set to `127.0.0.1`, did run, but with that setting my computer lost `sdb` access. If you try apps like these, expect trial and error.

**Can I remove a sideloaded app?**
Yes: `tizen uninstall -p PACKAGE_ID -t YOUR_TV_MODEL`, or hold the Enter button on the app in the TV's app list.

---

Questions or fixes welcome in the issues.
