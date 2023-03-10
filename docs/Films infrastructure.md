# Films Infrastructure

This is a guide for setting up films infra after running [films playbook](../playbooks/films.yml).

## Qbitttorent

Open `host:6004` to and enter with default credentials (user: `admin`, password: `adminadmin`).
- In options change `Default Save Path` to `/media`.
- Change admin password.
- `BitTorrent > Seeding limits` enable limit by time with option `remove torrent and its files` to automatically delete downloaded files after radarr it imports to the library.

## Jackett

Open `host:5999` and add some indexers. Copy `Torznab feed` for added indexers.

## Radarr

Open `host:8002`.
- Settings > Indexers: add torznab indexers from jackett. (change url to `jackett.films_net`)
- Settings > Download clients: 
    - add qbittorrent
    - host: torrent.films_net
    - port: 6004
    - username and password
- Settigns > General: change authentication to forms and specify credentials.
- Settings > Media manager: `show advansed settings` and enable `Rename movies`.

## Sonarr

Do same things as for Radarr, but on the `host:8003`.

## Jellyfin

Open `host:8096` and go through the initial setup.
- Add medialibrary with `/media` folder.

## Restart all

Run this command to stop docker containers:
```sh
docker kill jellyfin qbittorrent jackett radarr
docker system prune
```

Rerun ansible script.
