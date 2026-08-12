<p align="center">
  <a href="https://github.com/epoplavskis/homeassistant_salus"><img src="https://shop.salusinc.com/cdn/shop/files/saluslogo_-_600x160_eea8b0d2-55bf-43fa-a46a-2455b56d73c5_384x128.png?v=1623083270" height="140"></a>
</p>

# HomeAssistant - Salus Controls iT600 Smart Home Custom Component

> **Note:** this is a personal fork of
> [epoplavskis/homeassistant_salus](https://github.com/epoplavskis/homeassistant_salus).
> The only change is that a gateway that is unreachable or not finished booting
> now raises `ConfigEntryNotReady` instead of failing the setup outright, so
> Home Assistant retries on its own. See [Changes in this fork](#changes-in-this-fork).

# What This Is

This is a custom component to allows you to control and monitor your Salus iT600 smart home devices locally through Salus Controls UGE600 / UGE600 universal gateway.

# Supported devices

See the [readme of underlying pyit600 library](https://github.com/epoplavskis/pyit600/blob/master/README.md)

# Installation and Configuration

## HACS (recommended)

This card is available in [HACS](https://hacs.xyz/) (Home Assistant Community Store).
*HACS is a third party community store and is not included in Home Assistant out of the box.*

## Manual install
Copy `custom_components` folder from this repository to `/config` of your Home Assistant instalation.

To configure this integration, go to Home Assistant web interface Configuration -> Integrations and then press "+" button and select "Salus iT600".

When you are done with configuration you should see your devices in Configuration -> Integrations -> Entities

# Changes in this fork

## 0.5.4 - retry setup instead of giving up

Upstream, a gateway that could not be reached or that rejected the EUID logged
an error and returned `False` from `async_setup_entry`. Home Assistant treats
that as a permanent setup failure: the entry is marked `setup_error` and is
never retried, so the integration stays dead until it is reloaded by hand.

That is the wrong outcome after a power cut. Home Assistant boots faster than
the UGE600 gateway, which also has to bring up its Zigbee network. While it is
still starting the gateway answers with data that cannot be decrypted, which
`pyit600` surfaces as `IT600AuthenticationError` - so a perfectly correct EUID
produced `Authentication error: check if you have specified gateway's EUID
correctly.` and all entities went permanently unavailable.

Both error paths now raise `ConfigEntryNotReady`, which is the documented way
for an integration to say "not yet". Home Assistant then retries with a
backoff until the gateway answers, and the entry recovers unattended.

# Troubleshooting

If you can't connect using EUID written down on the bottom of your gateway (which looks something like `001E5E0D32906128`), try using `0000000000000000` as EUID.

Also check if you have "Local Wifi Mode" enabled:
* Open Smart Home app on your phone
* Sign in
* Double tap your Gateway to open info screen
* Press gear icon to enter configuration
* Scroll down a bit and check if "Disable Local WiFi Mode" is set to "No"
* Scroll all the way down and save settings
* Restart Gateway by unplugging/plugging USB power
