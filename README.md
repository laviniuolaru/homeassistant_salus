<p align="center">
  <a href="https://github.com/epoplavskis/homeassistant_salus"><img src="https://shop.salusinc.com/cdn/shop/files/saluslogo_-_600x160_eea8b0d2-55bf-43fa-a46a-2455b56d73c5_384x128.png?v=1623083270" height="140"></a>
</p>

# HomeAssistant - Salus Controls iT600 Smart Home Custom Component

> **Note:** this is a personal fork of
> [epoplavskis/homeassistant_salus](https://github.com/epoplavskis/homeassistant_salus).
> The only change is in how gateway errors are classified: a gateway that is
> unreachable or still booting now raises `ConfigEntryNotReady` so Home
> Assistant retries on its own, while a reply that cannot be decrypted - a
> wrong EUID - raises `ConfigEntryError` and fails for good.
> See [Changes in this fork](#changes-in-this-fork).

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

## 0.5.5 - tell "still booting" apart from "wrong EUID"

0.5.4 retried on every error, which fixed the power-cut case but meant a
genuinely wrong EUID would also be retried forever instead of being reported.
Both cases can be separated exactly, so neither needs to be a compromise.

`IT600AuthenticationError` is a misnomer. `pyit600` raises it from one place
only, in `IT600Gateway.connect()`: the encrypted `POST /deviceid/read` failed
with a timeout or a connection error, *and* a plain `GET /` on the same host
still answered. The gateway's web server being up while its device API is not
serving yet is what a gateway that has not finished booting looks like - so
this is a transient condition, not a credentials problem.

A wrong EUID never reaches that branch. Measured against a UGE600:

| request | response | result |
| --- | --- | --- |
| correct EUID | HTTP 200, 19952 bytes, 64 ms | decrypts, `status: success` |
| wrong EUID | HTTP 200, 32 bytes, 10 ms | `ValueError: Invalid padding bytes` |

The gateway answers a request it cannot decrypt immediately, and the reply
fails to decrypt on the way back. `pyit600` catches that in the bare
`except Exception` of `_make_encrypted_request` and raises
`IT600CommandError`, which this integration did not catch at all - it escaped
as an unhandled traceback.

So the three errors now map to the three outcomes Home Assistant provides:

| pyit600 error | what it means | Home Assistant |
| --- | --- | --- |
| `IT600ConnectionError` | gateway unreachable | `ConfigEntryNotReady` - retry |
| `IT600AuthenticationError` | web server up, device API not serving yet | `ConfigEntryNotReady` - retry |
| `IT600CommandError` | reply will not decrypt, i.e. wrong EUID | `ConfigEntryError` - permanent |

## 0.5.4 - retry setup instead of giving up

Upstream, a gateway that could not be reached or that rejected the EUID logged
an error and returned `False` from `async_setup_entry`. Home Assistant treats
that as a permanent setup failure: the entry is marked `setup_error` and is
never retried, so the integration stays dead until it is reloaded by hand.

That is the wrong outcome after a power cut. Home Assistant boots faster than
the UGE600 gateway, which also has to bring up its Zigbee network, and a
gateway in that state is reported as `IT600AuthenticationError` - so a
perfectly correct EUID produced `Authentication error: check if you have
specified gateway's EUID correctly.` and all entities went permanently
unavailable.

Both error paths now raise `ConfigEntryNotReady`, which is the documented way
for an integration to say "not yet". Home Assistant then retries with a
backoff until the gateway answers, and the entry recovers unattended.

(This release assumed `IT600AuthenticationError` could also mean a wrong EUID,
and accepted retrying forever in that case as the cost. 0.5.5 above shows it
cannot, and drops the tradeoff.)

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
