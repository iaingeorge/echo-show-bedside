# Echo Show 5 Bedside Controller for Home Assistant

Turn a 2nd-gen Amazon Echo Show 5 into a bedside Home Assistant display: a big clock with the weather, one-tap light controls, a front-door lock status tile, and a baby monitor with sound from a UniFi Protect camera. It switches to a dim red night mode when the room goes dark.

This project was inspired by [The Stock Pot's video](https://youtu.be/6p5wvVl957c), which showed the build and the enclosure. This repo is the Home Assistant code side: the dashboard, scripts and automations, plus the problems I hit and how I solved them.

| Home | Baby (mockup) | Lights (mockup) |
|---|---|---|
| ![Home view: clock, date and weather, with the navigation rail on the right](docs/home.png) | ![Baby view mockup: full-height camera feed with the Baby button highlighted](docs/baby.png) | ![Lights view mockup: bedside lamp tiles, a Front Door lock tile, and On / Low / Off for the child's room light](docs/lights.png) |

*The Baby and Lights images are mockups rendered from the dashboard's styles. The camera picture is an illustration of a camera feed.*

## What you get

| View | What it shows |
|---|---|
| **Home** | Large clock, date and current weather over a background image |
| **Baby** | Full-height live camera with audio; tap the video for the mute button |
| **Lights** | Big tiles for the bedside lamps, a **Front Door** tile (locked/unlocked and door open/closed), and On / Low (dim red) / Off for the child's room light |

A navigation rail on the right of every view has Home, Baby and Lights buttons, plus quick toggles for the child's room and bedroom lights.

**Night mode.** When the Echo's light sensor reads 5 lux or less, every view switches to a dim red palette that won't light up the room.

**Front door check from bed.** The Front Door tile shows whether the door is locked and whether it's open or closed: green when locked, amber when unlocked, red when the door is open. Tap to lock (it won't lock an open door); long-press to unlock, with a confirmation prompt. A companion automation locks the door every night and tells you if it couldn't.

**The baby monitor stays open.** View Assist normally goes back to its default view after a timeout. An automation sends the Echo back to the camera while the monitor is "on".

## Hardware

- Amazon Echo Show 5, **2nd gen** (model C76N8S), jailbroken and running **LineageOS** (TWRP kept)
- The enclosure from The Stock Pot: [build instructions on Printables](https://www.printables.com/model/1769972-amazon-echo-show-enclosure), or buy it ready-made from [his website](https://www.thestockpot.net/).
- A camera in Home Assistant. I used a UniFi Protect camera, but any `camera.*` entity should work.
- *Optional:* [Amazon Ethernet Adapter for Fire TV devices](https://www.amazon.com/dp/B074TC662N) (micro-USB) to put the Echo on wired Ethernet. See [Wired Ethernet instead of Wi-Fi](#wired-ethernet-instead-of-wi-fi).

> **Jailbreak and LineageOS.** I followed these two, for the **Echo Show 5 2nd gen (2021, codename "cronos")**:
>
> - [Video walkthrough of the jailbreak and the LineageOS install](https://youtu.be/cuPikKRJQ2I)
> - [XDA: Unlock / Root / TWRP / Unbrick Amazon Echo Show 5 2nd Gen (2021) – cronos](https://xdaforums.com/t/unlock-root-twrp-unbrick-amazon-echo-show-5-2nd-gen-2021-cronos.4772596/)
>
> ⚠️ These steps are for the **2nd gen only**. The 1st gen Echo Show 5 is different hardware and needs different instructions. Check your model before you start.

## Software on the Echo

- **[View Assist Companion App (VACA)](https://github.com/msp1974/ViewAssist_Companion_App)**, which shows the dashboard and exposes the Echo to Home Assistant (light sensor, screen brightness, media player, wake word and more)

## Home Assistant requirements

- **[View Assist](https://github.com/dinki/view_assist_integration)** integration ([docs](https://dinki.github.io/View-Assist/)), for `view_assist.navigate` and the `<viewassist-clock>` element
- **VACA** integration (comes with the companion app)
- Frontend cards from HACS / the Marketplace:
  - [button-card](https://github.com/custom-cards/button-card): used by this dashboard and required by View Assist
  - [card-mod](https://github.com/thomasloven/lovelace-card-mod): required by View Assist ([its prerequisites](https://dinki.github.io/View-Assist/docs/viewassist-setup/homeassistant-configuration/viewassist-configuration/dashboard-views/lovelace-prereqs))
  - [layout-card](https://github.com/thomasloven/lovelace-layout-card): required by View Assist
  - [Advanced Camera Card](https://github.com/dermotduffy/advanced-camera-card): the Baby view
- Tested on Home Assistant 2026.11

## Setup

### 1. Create the helper

Settings → Devices & services → Helpers → Create helper → **Toggle**

- Name: `Bedside monitor`
- Entity ID: `input_boolean.bedside_monitor`

The scripts and the "keep baby monitor open" automation use it.

For the nightly auto-lock (optional), also create a **Date and/or time** helper set to **Time** only:

- Name: `Front Door Auto-Lock Time`
- Entity ID: `input_datetime.front_door_auto_lock_time`
- Set it to your usual bedtime, for example 10:00 PM. You can change it any time.

### 2. Set up View Assist for the Echo

If you haven't already, follow the [View Assist](https://dinki.github.io/View-Assist/) and [VACA](https://dinki.github.io/View-Assist/docs/viewassist-setup/display-devices/vaca) setup guides so the Echo shows up in Home Assistant. Install **both** the VACA app on the Echo **and** the VACA integration in Home Assistant.

Then add a View Assist device for the Echo (Settings → Devices & services → View Assist → Add device), choosing the Echo as its display. In that device's settings, point it at this dashboard:

- Dashboard: `/dashboard-bedside`
- Home screen: `/dashboard-bedside/home`

(Field names can differ between View Assist versions.) View Assist creates a sensor for the device, such as `sensor.bedside_echo`. That's the entity the scripts and automation use to navigate the Echo.

### 3. Map the entities

Every entity ID in this repo is a placeholder. Search and replace them with your own:

| Placeholder | What it is |
|---|---|
| `sensor.bedside_echo_light_level` | The Echo's light-level sensor (VACA). Drives night mode. |
| `sensor.bedside_echo` | The Echo's View Assist device sensor (step 2), used for navigation |
| `weather.home` | Any weather entity |
| `camera.nursery` | The camera for the Baby view |
| `light.nursery` | Child's room light (rail toggle and On / Low / Off) |
| `light.bedroom` | Main bedroom light (rail toggle) |
| `light.bedside_left`, `light.bedside_right` | Bedside lamps |
| `lock.front_door` | Front door lock |
| `binary_sensor.front_door` | The lock's own door sensor (open/closed) |
| `binary_sensor.front_door_contact` | A separate contact sensor on the same door |
| `notify.mobile_app_phone_1`, `notify.mobile_app_phone_2` | Phones for auto-lock notifications |

**About the door sensors.** The tile and the auto-lock treat the door as open if *either* sensor says open. I use both because my lock's built-in sensor and a separate contact sensor each drop out now and then. If you only have one, delete the `binary_sensor.front_door_contact` lines. A sensor that's offline counts as "not open", so a flaky sensor doesn't stop the door locking every night.

**About the child's room light.** The **On** button sets a warm white (`color_temp_kelvin: 2700`) and **Low** sets dim red (`rgb_color: [255, 0, 0]`), so this needs a colour bulb. For a plain dimmable bulb, change those two buttons in the dashboard to use `brightness_pct` only.

**About the camera.** For sound on the Baby view, the camera's own microphone must be on (for UniFi Protect, in the camera's settings in the Protect app).

### 4. Add the dashboard

1. Settings → Dashboards → **Add dashboard** → "New dashboard from scratch". Give it the URL `dashboard-bedside`, or change the paths in the scripts and automation to match.
2. Open it → ⋮ → Edit dashboard → ⋮ → **Raw configuration editor**.
3. Paste in [`dashboard/bedside-dashboard.yaml`](dashboard/bedside-dashboard.yaml) and save.

The Home view's daytime background is `/view_assist/dashboard/background.jpg`, one of the images that comes with View Assist's own dashboard. If it's missing (the Home view has no background), install View Assist's dashboard and views, or change that path in the `page` card's `background` to any image you like, for example one in `/local/`.

### 5. Add the scripts and automation

- [`scripts.yaml`](scripts.yaml): `bedside_home`, `bedside_baby` and `bedside_lights`, used by the rail buttons, and `front_door_lock_if_closed`, used by the Front Door tile
- [`automations.yaml`](automations.yaml): keeps the baby monitor open, and the optional **Front Door - Nightly Auto-Lock**

**Keep the script IDs exactly as they are.** The rail buttons call `script.bedside_home`, `script.bedside_baby` and `script.bedside_lights`, and the Front Door tile calls `script.front_door_lock_if_closed`. If you create them in the script editor, Home Assistant builds the ID from the name you type, so open each one in YAML mode and check the ID matches. Otherwise the buttons do nothing.

### 6. Recommended VACA settings

Set these in the VACA app on the Echo, or with VACA's entities in Home Assistant:

| Setting | Value | Why |
|---|---|---|
| **Wake word** | **None** | **Important.** With a wake word on, the baby monitor sound cuts out every few seconds. See [Lessons learned](#baby-monitor-audio-played-1-second-out-of-every-7). |
| Screen always on | On | The clock stays visible |
| Auto brightness | On | Dims the screen with the room light, so no automation is needed. Night mode only changes the dashboard's colours, so the two work together. |
| Dark mode | On | Dark app chrome around the dashboard |
| Show status bar | Off | Full screen |
| Screen saver | Off | Would hide the clock |

### 7. Recommended LineageOS settings

These make the Echo behave like an appliance: it boots straight into the dashboard and stays there.

- **Make VACA the home app (launcher).** On Android 10 and later, this is how VACA starts on boot (it replaced its old "start on boot" option). After a power cut, the Echo comes back to the dashboard on its own. Set it when VACA prompts you, or in Settings → Apps → Default apps → Home app.
- **Turn off the lock screen.** Settings → Security → Screen lock → **None**, so a reboot doesn't stop at a lock screen.
- **Turn off battery optimisation for VACA.** Settings → Apps → VACA → Battery → **Not optimized** / Unrestricted, so Android never puts it to sleep in the background.
- **Sign in once.** VACA keeps its Home Assistant login after a reboot, so there's no auto-login to set up.
- **Optional backup for the screen:** Settings → System → Developer options → **Stay awake** (while charging). VACA's "Screen always on" already handles this, but it does no harm.

### Tweaks

- **Night mode threshold:** night mode turns on at **5 lux** or less. To change it, edit the `lux <= 5` line in the `night_vars` template at the top of the dashboard.
- **Section headings:** the Lights view's headings ("Master Bedroom", "Front Door" and "Child") are the `name:` values of the `section_title` cards.
- **No lock?** Delete the `dt` and `lk` cards from the Lights view and change the first two grid rows back to `"bt bt bt bt bt bt"` and `"l l l r r r"`.

## Lessons learned (the useful part)

### The dashboard couldn't be edited anymore

The `page` template uses `position: fixed` to fill the whole screen and hide Home Assistant's header and sidebar. That also covers the dashboard editor. The fix is to make it switch on the URL:

```yaml
- position: "[[[ return window.location.search.includes('edit=1') ? 'relative' : 'fixed'; ]]]"
```

Open `/dashboard-bedside/home?edit=1` on a computer to edit. The Echo never uses that parameter, so it always gets full screen.

### Baby monitor audio played 1 second out of every 7

This one took the longest. The video was smooth, but the sound came through in short bursts. Switching players (WebRTC vs Home Assistant's own player), switching streams (low vs medium) and even playing the stream on the Echo's native media player all gave the same 7-second cycle.

**Cause: the VACA wake word.** It keeps the microphone open, and on the Echo that keeps interrupting media audio. Setting the wake word to **None** (VACA's `Wake word` select entity, or in the app) fixed it right away. I don't use voice on this device, so I left it off. If you want voice, pause the wake word while `input_boolean.bedside_monitor` is on and restore it when it turns off.

### Which camera player and stream

- `live_provider: ha` (Home Assistant's own player) worked much better on the Echo than the WebRTC card.
- Once the wake word was off, the medium-resolution stream ran fine. UniFi Protect also offers a low-resolution channel to fall back on (it may be disabled by default in HA).

### Grey bar over the video

Advanced Camera Card's `menu.style: overlay` draws a grey bar over the video even when only the mute button is enabled, and overriding the background colour didn't work for me. `menu.style: hidden` leaves the video clean, and tapping the video brings up the mute button.

### Filling the screen

The Echo Show 5 is 960×480. Sizing the camera to the full height at 16:9 and pinning it to the left fills the space next to the rail:

```yaml
cam:
  - width: calc(100vh * 16 / 9)
  - max-width: 100%
  - justify-self: start
```

### The nightly auto-lock: things that bit me

The first idea was "lock the door at 10pm". It turned into a bit more than that:

- **Don't lock an open door.** A smart lock will happily throw its bolt into thin air, and then the door won't close. Both the tile and the automation check the door sensors first; the automation waits up to 10 minutes for the door to close, and notifies you if it doesn't.
- **The lock goes offline.** Mine dropped off the network many times a week. The automation waits a few minutes for it to come back, then tries again at midnight and 2am.
- **"Locked" is a state, not a command.** It sends `lock.lock`, then waits for the lock to *report* locked, up to 3 tries, and tells you if it never does.
- **Late-night trips outside.** If someone unlocks the door after bedtime (taking the bins out) and forgets, it relocks after 10 minutes.
- **Unlocking from the bedside should be deliberate.** Tap only locks; unlocking needs a long-press and a confirmation.

### Wired Ethernet instead of Wi-Fi

Our Wi-Fi is unreliable, and an always-on camera stream makes that obvious. I put the Echo on wired Ethernet with Amazon's [micro-USB Ethernet adapter for Fire TV devices](https://www.amazon.com/dp/B074TC662N), and it's been more stable since. Your mileage may vary: if your Wi-Fi is solid, you may not need it.

## Credits

- [The Stock Pot](https://www.thestockpot.net/) ([video](https://youtu.be/6p5wvVl957c), [Printables](https://www.printables.com/model/1769972-amazon-echo-show-enclosure)) for the original idea and the enclosure
- The View Assist and VACA developers
- The authors of button-card and Advanced Camera Card

## License

[MIT](LICENSE). Use it, change it and share it; just keep the copyright notice.
