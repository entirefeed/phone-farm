# What breaks in a phone farm

Find the symptom, run the check, then apply the fix. Replace `SERIAL` with the phone's serial number.

## Connection

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| One phone is missing from `adb devices` | Cable, port or the phone itself | `lsusb -t` to see if the host still sees a USB device | Swap the cable, then the port. Power-cycle the port with `sudo uhubctl -l HUB -p PORT -a cycle` |
| A group of phones disappears together | The hub they share, its power supply, or the USB controller behind it | `dmesg` on the host for disconnect messages; `lsusb -t` to see what they share | Reseat or replace the hub. Check the hub's power supply rating against the number of phones charging |
| Phones stop appearing before the ports are full | The controller's endpoint limit | Count devices per controller in `lsusb -t` | Move a hub to a port on another controller, or add a USB card. See the [Cambrionix note on endpoints](https://knowledge.cambrionix.com/Content/Technical-Notes/Endpoints.htm) |
| State is `unauthorized` | The phone has not accepted this host's RSA key | Look at the phone's screen through a camera or in person | Accept the prompt with "always allow". This needs a tap on the phone |
| State is `offline` | adb lost the session; common after a reboot or a system update | `adb -s SERIAL get-state` | `adb kill-server` then `adb start-server`. If it stays offline, power-cycle the port |
| Every phone is offline at once | The adb server on the host | `adb devices` hangs or prints nothing | Restart the adb server. Make sure only one adb version is installed on the host |

## Automation

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| Script fails on a button that used to work | The app updated and the screen changed | `adb -s SERIAL shell dumpsys package com.example.app \| grep versionName` | Turn off automatic updates. Update one phone, fix the selectors, then update the rest |
| Script fails on one phone only | A different language, model or Android version | `adb -s SERIAL shell getprop ro.product.model` and compare with the inventory | Make the phones the same, or select by resource ID and not by text |
| Script taps the wrong thing | A dialog is on top of the app | `adb -s SERIAL shell dumpsys window \| grep mCurrentFocus`, or a screenshot | Dismiss known dialogs in one shared function before each step |
| Script works by hand and fails in a job | A fixed sleep that is too short under load | Compare timings with 1 phone and with all phones running | Wait for elements, never for a fixed time |
| The same action happened twice | An automatic retry after a lost connection | Job history: did the first attempt reach the action? | Mark such jobs "unknown" and check the result before any retry |

## Hardware

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| Phone is slow, charging slowly or Wi-Fi turns off | Heat | `adb -s SERIAL shell dumpsys battery \| grep temperature` (tenths of a degree Celsius) | More space and airflow. Lower the load on that shelf |
| Phone shuts down or restarts by itself | Heat or a worn battery | Battery level and temperature history | Replace the battery or the phone |
| Back cover lifts or the screen bulges | Swollen battery | Look at the phone | Take it out of service now. Do not charge it. Dispose of it as battery waste |
| Screen stays black in scrcpy | The phone is asleep or locked | `adb -s SERIAL shell dumpsys power \| grep mWakefulness` | `adb -s SERIAL shell input keyevent KEYCODE_WAKEUP`. Set stay-awake as in the setup guide |

## Jobs

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| "No free phones" while phones sit idle | Reservations left by jobs that crashed | List reservations older than their time limit | Give every reservation an expiry and release expired ones |
| Jobs are marked done but the result is missing | The job trusted the phone's own screen | Compare "done" with the outside check | Add the outside check and the "unknown" status. See [architecture](architecture.md#job-statuses) |
| Uploads are slow in the evening | Wi-Fi shared by too many phones | Throughput per phone with all phones active | Add an access point, or give phones the host's wired connection |

## When a platform restricts an account

That is the platform applying its rules. It is not a hardware fault, and nothing in this repository is meant to get around it. Read the platform's rules, stop the job that caused it, and ask the platform through its own support channel.
