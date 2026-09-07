# Plant Butler

A hobby system that waters house plants. Three parts:

- **Board** (`firmware`): an Arduino UNO R4 WiFi with one soil-moisture sensor per pot, a pump, and
  a manifold that sends the water to one pot at a time. It reports readings and waters on command.
- **Backend** (`backend`): a small Python service in Docker on a Synology NAS (a network-attached
  storage box, the home server). It stores the readings, decides when to water, and sends alerts.
- **App** (`app`): an Android app to look at the plants, edit them, and water by hand.

This repository holds no code. It pins the five repositories below as git submodules and keeps the
documents that belong to the whole: this README, [DECISIONS.md](DECISIONS.md) (why things are the
way they are) and [GLOSSARY.md](GLOSSARY.md) (what the words mean).

## How the parts talk

```
 sensors, float,        ARDUINO UNO R4 WIFI               SYNOLOGY NAS                ANDROID PHONE
 meter, pump,  <---->   firmware                          backend (Docker)            app
 manifold                  |                                  |                          |
                           |  POST /report  c=0 ch0=8123 ... |                          |
                           | -------------------------------> |                          |
                           |  next=60  [+ at most 1 command]  |   HTTP, same token       |
                           | <------------------------------- | <----------------------> |
                                                              |
                                                          ntfy.sh (alerts to the phone)
```

The board talks first. Every report interval it posts one line of `key=value` pairs (called
`k=v` in these repositories) with a shared token in the `X-Token` header. The answer is `k=v` too:
the next interval and, when one is queued, one command such as "water outlet 3 with 50 ml". The
backend never calls the board. The app calls the backend over HTTP with the same token; the NAS is
reachable only on the home network or over Tailscale (a private network between your devices).
Watering logic lives in the backend; the board holds the safety limits and fails dry: when in
doubt, no water.

## Get started as a user

1. **Hardware.** Parts, printed pieces and wiring: [cad](cad/README.md).
2. **Backend.** Build and run the container on the NAS, choose a token: [backend](backend/README.md),
   section "Deploy".
3. **Board.** Put your WiFi and the token in `secrets.h`, build and flash:
   [firmware](firmware/README.md), section "Build and flash".
4. **App.** Build the APK (the Android install file), install it on the phone, and type the
   backend address and token on first start: [app](app/README.md), section "Install".

## Get started as a developer

```bash
git clone --recurse-submodules https://github.com/plantbutler/plantbutler.git
cd plantbutler
# if you cloned without the flag, or a submodule directory is empty:
git submodule update --init --recursive
```

A submodule is a pinned commit of another repository. `git pull` here moves the pins; run the
`submodule update` line afterwards to move the checkouts with them. When you push a change in a
submodule, commit the new pin here too, or the umbrella describes a state nobody can reproduce.

| repository | what it is | state |
| --- | --- | --- |
| [firmware](firmware/) | PlatformIO project for the board, C++ | bench-tested on the real rig |
| [backend](backend/) | Python service, SQLite, one container | 0.20.0 running on the NAS |
| [app](app/) | Android, Kotlin and Jetpack Compose | installed on the phone |
| [cad](cad/) | OpenSCAD parts, wiring drawings, parts list | in progress |
| [plan](plan/) | what is being built and in what order | see its README |

Each repository's README says how to build, test and run it and lists its files. Each `AGENTS.md`
holds the rules an automated agent (or a careful human) follows when editing there.

Before changing how the parts fit together, read [DECISIONS.md](DECISIONS.md): it has an index at
the top. Two rules hold everywhere: a decision is never edited, only followed by a newer one; and
secrets (WiFi passwords, the token, the NAS address) never enter a repository.
