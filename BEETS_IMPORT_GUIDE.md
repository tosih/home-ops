# Beets Import Guide - What.CD Library

## Your Library Stats

- **Location:** `/media/Music/What.CD`
- **Albums:** ~2,294 folders
- **Tracks:** 31,391 FLAC files
- **Estimated Import Time:** 10-48 hours (depends on matching speed)

## Pre-Import Checklist

### 1. Fix Permissions (Important!)

Some folders have permission issues. Let's fix them:

```bash
# Shell into Beets
kubectl exec -it -n media deployment/beets -- bash

# Fix permissions on the What.CD folder
# This assumes the NFS share allows changing permissions
chmod -R 755 /media/Music/What.CD 2>/dev/null || echo "Permission fix may need to be done on NFS server"
```

If that doesn't work, you may need to fix permissions on your NFS server at `10.0.0.2:/mnt/data/media/Music/What.CD`

### 2. Test Import (Highly Recommended)

Before importing 2,000+ albums, test with one album:

```bash
kubectl exec -it -n media deployment/beets -- bash

# Test with one album
beet import "/media/Music/What.CD/The Shins - Wincing The Night Away"

# This will show you:
# - How Beets matches albums
# - What the organized structure looks like
# - How to answer prompts
```

## Import Strategies

### Strategy 1: Interactive Import (Most Accurate)

**Best for:** Ensuring perfect matches, reviewing edge cases  
**Time Required:** Days (you need to answer prompts)

```bash
kubectl exec -it -n media deployment/beets -- bash

# Install screen to keep session alive
apk add screen

# Start screen session
screen -S beets-import

# Run interactive import
beet import /media/Music/What.CD

# Detach: Ctrl+A then D
# Reattach: screen -r beets-import
```

### Strategy 2: Quiet Mode (Recommended for Large Libraries)

**Best for:** Auto-accepting strong matches, only reviewing uncertain ones  
**Time Required:** 12-48 hours (mostly automated)

```bash
kubectl exec -it -n media deployment/beets -- bash
apk add screen
screen -S beets-import

# Quiet mode: auto-accepts matches >90% similarity
beet import -q /media/Music/What.CD

# Detach: Ctrl+A then D
```

### Strategy 3: Incremental Import (Safest)

**Best for:** Testing and gradual migration  
**Time Required:** Weeks (but safest approach)

Import by letter or small batches:

```bash
# Import artists starting with 'A'
beet import -q "/media/Music/What.CD/A*"

# Import artists starting with 'B'
beet import -q "/media/Music/What.CD/B*"

# etc...
```

### Strategy 4: Very Quiet Mode (Fastest, Least Accurate)

**Use only if:** You trust your existing metadata  
**Time Required:** 6-12 hours

```bash
# Auto-accepts everything, even low similarity
beet import -q -t /media/Music/What.CD

# WARNING: May create incorrect matches!
```

## Recommended Approach for Your Library

Given you have What.CD rips (typically well-tagged), I recommend:

### Phase 1: Test (Now)
```bash
kubectl exec -it -n media deployment/beets -- bash

# Test one album
beet import "/media/Music/What.CD/The Shins - Wincing The Night Away"

# Check the result
ls -la /media/music/
```

### Phase 2: Small Batch (Today)
```bash
# Import first 50 albums to verify process
beet import -q "/media/Music/What.CD/[A-C]*"
```

### Phase 3: Full Import (Overnight/Weekend)
```bash
apk add screen
screen -S beets-import

# Start the full import in quiet mode
beet import -q /media/Music/What.CD > /config/import.log 2>&1

# Detach: Ctrl+A then D
```

### Phase 4: Review (After Import)
```bash
# Check for albums that couldn't be matched
beet ls -a singleton:true

# Check duplicates
beet duplicates

# View import log
cat /config/import.log
```

## During Import

### Monitor Progress

```bash
# Check Beets logs
kubectl logs -n media deployment/beets -f

# Check how many albums imported so far
kubectl exec -n media deployment/beets -- beet stats

# Reattach to screen session
kubectl exec -it -n media deployment/beets -- screen -r beets-import
```

### Import Prompts

When Beets finds a match, you'll see:

```
Artist - Album (2020)
(Similarity: 95.2%)
[A]pply, More candidates, Skip, Use as-is, as Tracks, Enter search, aBort?
```

**Common responses:**
- `A` - Apply this match (use this for 90%+ matches)
- `S` - Skip this album
- `U` - Use as-is (keep existing metadata)
- `M` - Show more candidates
- `I` - Enter MusicBrainz ID manually

### Expected Timeline

**Quiet mode (`-q`) estimate:**
- ~10-20 seconds per album (on average)
- 2,294 albums × 15 seconds = ~9.5 hours
- **Real time:** 12-24 hours (accounting for network, MusicBrainz API limits)

## After Import

### 1. Verify Organization

```bash
kubectl exec -n media deployment/beets -- bash

# Check organized library
ls -la /media/music/

# Get stats
beet stats

# Sample output:
# Tracks: 31391
# Total time: 98.5 days
# Approximate total size: 500 GB
# Artists: 1200
# Albums: 2294
```

### 2. Update Plex

1. **Plex Settings** → **Libraries** → **Music**
2. **Edit Library** or **Add Library**:
   - Folder: `/media/music`
   - Scanner: Plex Music
   - Agent: Plex Music
   - ✅ Use embedded tags
3. **Scan Library**

### 3. Keep Original Files (Optional)

After verifying the organized library works in Plex:

```bash
# Your original files are still at:
/media/Music/What.CD

# Beets MOVED them to:
/media/music/Artist/Album/Tracks.flac

# The What.CD folder should now be empty or nearly empty
# (Some files may remain if they couldn't be matched)
```

### 4. Handle Unmatched Files

```bash
# Find albums that couldn't be matched
beet ls -a singleton:true

# These are stored in:
/media/music/Non-Album/

# You can:
# - Manually import them later
# - Fix tags and re-import
# - Leave them as-is
```

## Troubleshooting

### Permission Denied Errors

```bash
# Fix from NFS server at 10.0.0.2
ssh admin@10.0.0.2
chmod -R 755 /mnt/data/media/Music/What.CD
chown -R 568:568 /mnt/data/media/Music/What.CD
```

### Slow Import Speed

```bash
# MusicBrainz API rate limiting is normal
# You can't speed this up much, but you can:
# - Use -q flag (reduces prompts)
# - Import in smaller batches
# - Be patient (it's worth it!)
```

### Failed Matches

```bash
# If many albums fail to match:
# 1. Check internet connectivity
# 2. Verify MusicBrainz is accessible
# 3. Try manual search with 'I' option
# 4. Use 'U' to keep existing metadata
```

### Out of Disk Space

```bash
# Check available space
kubectl exec -n media deployment/beets -- df -h /media

# Beets MOVES files, so total space used should be the same
# If you're low on space, import in batches
```

## Commands Reference

```bash
# Access Beets container
kubectl exec -it -n media deployment/beets -- bash

# Start import
beet import -q /media/Music/What.CD

# Check progress
beet stats

# View logs
cat /config/beet.log

# List imported albums
beet ls -a

# Find duplicates
beet duplicates

# Update metadata
beet update

# Fetch missing art
beet fetchart
```

## Next Steps

1. **Run test import** on one album
2. **Verify** the organized output in `/media/music`
3. **Start small batch** (50-100 albums)
4. **Check in Plex** to make sure it looks good
5. **Start full import** in screen session
6. **Wait 12-48 hours** for completion
7. **Update Plex** to use `/media/music`
8. **Enjoy** your perfectly organized library!

---

**Ready to start?** Run this:

```bash
kubectl exec -it -n media deployment/beets -- bash
```

Then paste:

```bash
# Test with one album first
beet import "/media/Music/What.CD/The Shins - Wincing The Night Away"
```
