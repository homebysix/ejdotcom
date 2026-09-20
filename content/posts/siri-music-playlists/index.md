---
title: Using Siri for programmatic creation of Music app playlists
date: 2026-09-20T12:34:26-0700
description: In macOS 27, Siri can produce scripts that can create playlists more complex than the normal Music app’s smart playlists feature supports.
slug: siri-music-playlists
tags:
    - python
    - music
---

My experience with the macOS 27 betas yielded thankfully little drama, so I made an uncharacteristic decision to install the public 27.0 release on my personal Mac when it became available last week. The new and improved Siri was one of the standout features, so I decided to see whether Siri would be newly useful for creating playlists in the Music app. What I discovered surprised me: when pushed beyond the basic smart playlist capabilities, Siri was able to generate a working Python script for playlist generation. It also introduced me to a Music app feature I had no idea existed — which is impressive given the cumulative amount of time I’ve spent in that app and its predecessor iTunes.

My goal was reasonably simple: I wanted to create a playlist which contained all my albums that have no “rated” songs. (This is sometimes an indicator that I haven’t given an album enough playtime to form an opinion about it.) The built-in smart playlists feature can’t do this, because it only evaluates criteria per song and doesn’t consider entire albums together.

When prompted with this goal, Siri’s first response suggested (among other things) a smart playlist using the “album rating” metadata. I only rate individual songs, not albums, so that wouldn’t work for me.

The Music app allows exporting your entire library to an xml file. This file includes the metadata for each track in your collection, including the rating. I knew it would be possible to write some Python that parses this file (as [GraphMyTunes](../graphmytunes) does) and generates my desired list of albums, but I wasn’t sure how to get that list back into the Music app itself. So I gave Siri this breadcrumb: “What about exporting the library as an xml file and using Python[^1] to parse it and create the playlist I'm describing?” This produced a simple Python script that correctly output the list of albums, along with this suggestion that caught me off guard:

> Once you have the list of unrated albums, you can write their file locations to an .m3u file and import that playlist back into your music library. Would you like more details on how to format the M3U playlist file for import?

I wasn’t aware that m3u files could be used to import playlists of files that were _already_ present in the library; I thought they were solely for importing _new_ files. Siri pointed me to the option in *File > Library > Import Playlist*.

I asked Siri to rewrite the script to include creating the m3u (technically m3u8) file, as well as excluding albums with fewer than 4 songs and incomplete albums (based on track number and total track metadata). The final script Siri generated is below:

```py
import plistlib
from urllib.parse import unquote, urlparse

# --- Configuration ---
LIBRARY_XML_PATH = "/path/to/music_library_export.xml"
PLAYLIST_FILENAME = "Unrated_Albums.m3u8"
MIN_SONGS_PER_ALBUM = 4
# ---------------------


def generate_playlist():
    with open(LIBRARY_XML_PATH, "rb") as f:
        library = plistlib.load(f)

    tracks = library["Tracks"]
    albums = {}

    # 1. Group tracks by album
    for track_id, track in tracks.items():
        album_name = track.get("Album", "Unknown Album")
        if album_name not in albums:
            albums[album_name] = []
        albums[album_name].append(track)

    unrated_song_paths = []

    # 2. Process each album
    for album_name, songs in albums.items():
        # Skip albums with fewer than the minimum required songs
        if len(songs) < MIN_SONGS_PER_ALBUM:
            continue

        has_rating = False
        track_numbers = set()
        total_tracks = 0

        for song in songs:
            if "Rating" in song:
                has_rating = True
                break

            if "Track Number" in song:
                track_numbers.add(song["Track Number"])
            if "Track Count" in song:
                total_tracks = max(total_tracks, song["Track Count"])

        if has_rating or total_tracks == 0 or len(track_numbers) == 0:
            continue

        # 3. Check for completeness
        is_complete = True
        for i in range(1, total_tracks + 1):
            if i not in track_numbers:
                is_complete = False
                break

        # 4. If complete and unrated, add to playlist
        if is_complete:
            for song in songs:
                if "Location" in song:
                    parsed_url = urlparse(song["Location"])
                    file_path = unquote(parsed_url.path)
                    unrated_song_paths.append(file_path)

    # 5. Write to M3U8
    with open(PLAYLIST_FILENAME, "w", encoding="utf-8") as playlist_file:
        playlist_file.write("#EXTM3U\n")
        for path in unrated_song_paths:
            playlist_file.write(f"{path}\n")

    print(
        f"Successfully created '{PLAYLIST_FILENAME}' with {len(unrated_song_paths)} songs."
    )


if __name__ == "__main__":
    generate_playlist()
```

There are some touch-ups I’d make to this script were I to keep maintaining it, but it’s impressive that this came directly from Siri. Even accounting for the nudge I gave it towards a Python solution, I was expecting it to need more guidance than it did. And it taught me about importing m3u playlists, which is an approach I might use more often now that I’m aware of it.

[^1]: I also unsuccessfully tried this experiment with AppleScript, hopeful that I could cut out the import step and create the playlist directly in the Music app. The AppleScript produced by Siri created an empty playlist, then stalled for multiple hours before I stopped it. So I’m content using Python here, especially because Python’s failure modes pose much less risk to my music library’s integrity.
