# Phone farm setup guide

This guide takes you from one Android phone on a desk to a small rack that runs jobs. It uses only open-source tools. Replace `SERIAL` with a phone's serial number and `com.example.app` with the app you are allowed to automate.

The order matters. Each step proves something the next step depends on.

## 1. Decide the task and the proof

Write two lines before you buy anything:

- **Task:** what one phone must do, for example "install our build and run the checkout flow" or "publish this approved video to our own account".
- **Proof:** what shows that it happened, for example a test report, a screenshot, or a public link that opens.

If you cannot name the proof, more phones will only produce more unknown results.

## 2. Make one phone work

Install [Android SDK Platform Tools](https://developer.android.com/tools/releases/platform-tools) on the host. On the phone, turn on Developer options, then USB debugging. Plug it in and accept the RSA key prompt on the phone's screen.

```bash
adb devices -l                                   # the phone should show the state "device"
adb -s SERIAL shell getprop ro.product.model     # which model is this
adb -s SERIAL shell getprop ro.build.version.release   # which Android version
adb -s SERIAL install -r app.apk                 # install or update your app
adb -s SERIAL push clip.mp4 /sdcard/Movies/clip.mp4    # copy a file to the phone
adb -s SERIAL exec-out screencap -p > shot.png   # take a screenshot
```

A file pushed with adb may not show in the phone's gallery until Android scans it. On many phones this asks for a scan, but it depends on the Android version, so test it on your model:

```bash
adb -s SERIAL shell am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE -d file:///sdcard/Movies/clip.mp4
```

See the screen with [scrcpy](https://github.com/Genymobile/scrcpy):

```bash
scrcpy -s SERIAL --max-size=1024 --no-audio
```

Do the whole task by hand through scrcpy once. If any part needs something the phone does not have, such as a camera on a board-only device, you have found it before buying twenty.

## 3. Prepare every phone the same way

Do these on each phone. Keep a checklist, because one phone that differs will fail in its own way.

- USB debugging on, and the host's RSA key accepted with "always allow".
- The same language on every phone. Scripts that find a button by its text break when one phone is in another language.
- Automatic app updates off, and system updates set to ask first. You choose when the screens change.
- The same app version everywhere. Check it:

```bash
adb -s SERIAL shell dumpsys package com.example.app | grep versionName
```

- Keep the screen awake while the phone is on power (3 means on AC or USB):

```bash
adb -s SERIAL shell settings put global stay_on_while_plugged_in 3
```

A phone with a PIN or pattern cannot be unlocked by a script. Most farms keep the phones in a locked room and set the screen lock to none. Decide that on purpose.

## 4. Connect the rack

- Use powered hubs. A hub that draws its power from the host cannot charge a row of phones.
- Prefer hubs that can switch power per port, and check them against the compatibility list in [uhubctl](https://github.com/mvp/uhubctl).
- Add phones a few at a time and watch `adb devices` after each group. If phones stop appearing before the ports are full, you have probably reached the USB controller's endpoint limit. Move a hub to another controller. Cambrionix explains endpoints [here](https://knowledge.cambrionix.com/Content/Technical-Notes/Endpoints.htm).
- See which hub and port each phone is on:

```bash
lsusb -t
sudo uhubctl
```

- Leave space between phones and move air across them. Read the battery temperature, which Android reports in tenths of a degree Celsius:

```bash
adb -s SERIAL shell dumpsys battery | grep -E "level|temperature"
```

## 5. Keep an inventory

A text file is enough to start. Put a printed label on each phone that matches the first column.

```csv
label,serial,model,android,hub,port,purpose,status
R1-01,SERIAL_OF_PHONE_1,Pixel 6,14,1-1.4,1,checkout tests,ok
R1-02,SERIAL_OF_PHONE_2,Pixel 6,14,1-1.4,2,checkout tests,ok
R1-03,SERIAL_OF_PHONE_3,Pixel 6,14,1-1.4,3,spare,battery replaced
```

List what is connected from Python with [adbutils](https://github.com/openatx/adbutils):

```python
import adbutils

for d in adbutils.adb.device_list():
    print(d.serial, d.prop.model)
```

Compare that output with the inventory on a timer. A phone that is in the file and missing from the list is your first alert.

## 6. Script the task

[uiautomator2](https://github.com/openatx/uiautomator2) is the shortest path from Python:

```bash
pip install uiautomator2
```

```python
import uiautomator2 as u2

d = u2.connect("SERIAL")
print(d.info)                      # screen size, Android version

d.app_start("com.example.app")

# Confirm the screen before acting on it.
if not d(text="Upload").wait(timeout=10):
    d.screenshot("unexpected.png")
    raise RuntimeError("Upload button not found: " + str(d.app_current()))

d(text="Upload").click()
d.screenshot("after-upload.png")
```

Rules that keep scripts alive:

1. Select by text or resource ID, never by screen position.
2. Wait for the element you need. Do not sleep for a fixed time.
3. Before every action, confirm you are on the screen you expect. If not, take a screenshot and stop.
4. Handle the dialogs you know about (permissions, update prompts) in one shared function.
5. Find selectors with [uiautodev](https://github.com/codeskyblue/uiautodev), or dump the screen:

```bash
adb -s SERIAL shell uiautomator dump /sdcard/ui.xml
adb -s SERIAL pull /sdcard/ui.xml
```

[Appium](https://github.com/appium/appium) and [Maestro](https://github.com/mobile-dev-inc/Maestro) do the same job with other trade-offs. Appium fits teams that already use WebDriver. Maestro fits short flows written in YAML.

## 7. Run jobs from a queue

Do not start scripts by hand or from cron with fixed phone numbers. Put jobs on a queue and let a worker on the host take them. Any queue in the [list](../README.md#queue-and-schedule-jobs) works.

A job needs at least: an ID, the task, the phone or the account it must run on, the input file, and a status.

A phone runs one job at a time, so the worker reserves it first. The reservation must expire, or a crashed job keeps the phone forever. With Redis that is one command, which here reserves phone R1-07 for 15 minutes only if nobody else holds it:

```text
SET phone:R1-07 job-123 NX EX 900
```

Before the job starts, the worker checks the phone:

```bash
adb -s SERIAL get-state                               # must print "device"
adb -s SERIAL shell dumpsys battery | grep -E "level|temperature"
adb -s SERIAL shell dumpsys window | grep mCurrentFocus   # which app is in front
```

## 8. Save evidence and check from outside

For every job, save the screenshot after the last step and whatever the app returns: a report, an ID or a link.

Then check the result from somewhere other than the phone. For a test, read the report. For a published post, open the public link from a server. The phone saying "done" and the result existing are two different facts.

Use three final statuses:

- **done:** the outside check passed.
- **failed:** the job stopped before the action, and nothing was sent.
- **unknown:** contact was lost after the action started. A person or a later check decides. Never retry an unknown job automatically, because the first attempt may have worked.

## 9. Monitor, then grow

Before adding phones, alert on these four things:

1. A phone in the inventory is not in `adb devices`.
2. A battery temperature is above the limit you chose.
3. A reservation is older than its time limit.
4. The share of jobs ending in "unknown" goes up.

Then add phones in small groups and watch the same numbers. The count that matters is jobs done and checked, not phones on the shelf.
