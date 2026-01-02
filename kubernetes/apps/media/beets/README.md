# Beets - Music Library Organizer

Beets organizes your FLAC music library with perfect metadata and a consistent structure that Plex loves.

## Architecture

- **Storage:** Shares the same `media-nfs` PVC as Plex
- **Source:** `/media` (your existing FLAC library)
- **Destination:** `/media/music` (organized library)
- **Web UI:** https://beets.tosih.org

## Initial Import

### Option 1: Web UI (Easier)
1. Visit https://beets.tosih.org
2. Browse your library
3. Use the web interface to manage imports

### Option 2: CLI (More Powerful)

Access the Beets container:
```bash
kubectl exec -it -n media deployment/beets -- bash
```

Import your existing music:
```bash
# Import from wherever your FLAC files currently are on /media
# For example, if they're in /media/flac:
beet import /media/flac

# Beets will:
# 1. Scan each album
# 2. Match against MusicBrainz
# 3. Ask for confirmation (or auto-import if confident)
# 4. Organize to /media/music with perfect structure
```

## Ongoing Usage

### Import New Music

When you add new music files to the NFS share:

```bash
# Shell into Beets
kubectl exec -it -n media deployment/beets -- bash

# Import new files
beet import /media/new-music
```

Or set up a CronJob (see below) to auto-import periodically.

## Beets Commands Reference

```bash
# Import music
beet import /path/to/music

# List your library
beet ls

# List albums
beet ls -a

# Search for specific artist
beet ls artist:"Artist Name"

# Find duplicates
beet duplicates

# Update metadata from MusicBrainz
beet update

# Check library stats
beet stats

# Fetch missing album art
beet fetchart

# Web UI (already running)
beet web
```

## File Structure

Beets organizes files like this:

```
/media/music/
├── Artist Name/
│   ├── Album Name (2023)/
│   │   ├── 01 Track Name.flac
│   │   ├── 02 Track Name.flac
│   │   └── cover.jpg
│   └── Another Album (2020)/
│       └── ...
└── Another Artist/
    └── ...
```

This is exactly what Plex expects!

## Plex Integration

After Beets organizes your library:

1. **Add Music Library in Plex:**
   - Type: Music
   - Folder: `/media/music`
   - Scanner: Plex Music
   - Agent: Plex Music
   - ✅ Enable "Use embedded tags"

2. **Plex will use:**
   - Beets' perfect metadata
   - Embedded album art
   - Organized folder structure
   - Clean file names

## Configuration

The Beets config is in the ConfigMap at:
`kubernetes/apps/media/beets/app/configmap.yaml`

Key settings:
- **Destination:** `/media/music`
- **Move files:** Yes (organizes from source to destination)
- **Write tags:** Yes (embeds metadata in FLAC files)
- **Plugins:** fetchart, embedart, duplicates, replaygain, lastgenre

## Automation (Optional)

To automatically import new music, you can create a CronJob:

```yaml
# Run beet import nightly
schedule: "0 2 * * *"  # 2 AM daily
command: beet import -q /media/incoming
```

## Troubleshooting

### Check logs
```bash
kubectl logs -n media deployment/beets -f
```

### View Beets log file
```bash
kubectl exec -it -n media deployment/beets -- cat /config/beet.log
```

### Database location
```bash
# Database is stored in PVC at:
/config/musiclibrary.db
```

### Reset/Reimport
```bash
# Clear database and start over
kubectl exec -it -n media deployment/beets -- bash
rm /config/musiclibrary.db
beet import /media/flac
```

## Resources

- [Beets Documentation](https://beets.readthedocs.io/)
- [Plugin Reference](https://beets.readthedocs.io/en/stable/plugins/index.html)
- [Path Formats](https://beets.readthedocs.io/en/stable/reference/pathformat.html)
