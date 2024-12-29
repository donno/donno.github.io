---
layout: post
title:  "End of Year 2024"
date:   2024-12-29 11:00:00 +1030
---
Closing out the year 2024. An overview on some things I did, played and watched
this year.

# Things I did for 2024

* Took some dancing lessons for first dance at wedding
* Got married
* Went on honeymoon to Seoul, JeJu and across Japan.
* Coding
    * Found some time to continue working in the world-building ideas for the game.
    I wasn't however able to find the time to write about the game, I might 
    try to get to writing at least one post during my remaining holidays early
    in the new year.
    * Expanded on the digital logic circuits project by adding NotGate and 
    adding a higher level concept such as integer to binary as well as a 
    Binary Coded Decimal to 7-segment Decoder.
    * Added support for basic styling to the vectortile renderer.
    * Wrote a tool for building graph of the dependencies of DLLs. This calls out
    to [Dependencies](0) by [lucasg](0) to find the dependencies of a set o
    DLLs then generates a graph in DOT language and runs GraphViz to render it
    to a PNG.
    * Created a proof-of-concept of a Python script which watches Notepad for
    keywords and the sends it to a HTTP service hosting llama.cpp and replaces
    the line with the result. Microsoft themselves are adding Copilot to it.
    * Started a small project to create a native PDF viewer built on [pdfium](2). 
    * Started using Rust sincerely for two projects.
        * One of them using the [openapi-generator][3] to handle creating a
          crate from an OpenAPI specification.\
          This was a neat exercise as the type system helps show where the
          specification not bounded enough and has extra states, i.e. it should
          make more guarantees on what is required and thus will always be
          provided.
        * One of them used [`ratatui`](4) while the other used 
          [`native-windows-gui`](5)
    * Built OCI images from Pytho
    * Tried out BitTorrent seeders and lechers in containers.
* Things I looked into
    * Setup Wireguard on my home network so I could connect to it while on
      public WiFi while in Japan.
    * IPXE / Netboot.xyz - Setting up ability to host Linux images on a HTTP
      server and move into them.
    * Iroh (a protocl for sychinig and moving bytes) using peer to pear network
       where blobs are opquee bytes verified by BLAKE3 hash.
    * Tile formats
        * MVT - Mapbox Vector Tile which are encoded with Google Protobufs, has the file extension mvf and mimetype application/vnd.mapbox-vector-tile.
        * MBTiles - A specification for storing tiled map data in SQLite databases for immediate usage and for transfer and used in a SQLite3 database. Can store raster (PNG) or vectors (MVT).
        * PMTiles - single-file archive format for tile data, designed to work with HTTP range requests
        * Related - Shortbread Vector Tiles, Mapbox Style, MapLibreStyle, Protomaps and VersaTiles
    * [Multi-Threaded Routing Toolkit (MRT)](6) Routing Information Export Format - didn't get far here.
    * Importing Jira data into [Taiga](7), [Plane](8) and [Huby](9).
    ** Taiga was the simplest.
    ** Plane has the problem you need all the users registered to be able to associate
    the issues with them and there no bulk.
    * Importing Atlassian Crucible data into GitLab as merge requests - this hit a
    big snag as GitLab doesn't do the heavy lifting here as when you create the
    reviews you need to duplicate the diffs rather than just point it at commits.

### Games

* Xbox Games Pass
    * Headbangers: Rhythm Royale - A new mode was introduced called
      "Battle of the Dancers" where it is a 1v1 mode that is similar to Guitar
      Hero and Rockband.\
      The game was removed from the Game Pass and while it was on Sale for
      around $5 on Steam I held back as I knew there were other games
      to play. I can see this game either going free-to-play or being shutdown
      in a year. As most matches were filled with bots for the royale mode.
    * It Takes Two - Started playing this with my Fiancée, completed the first two
      levels.
    * Palworld - I put about 48 hours into this before August 2024. The 
      resource collection, technology tree and building really hit the void from
      Minecraft as this year didn't play any Minecraft mods. The thing that got
      me behind was not capturing as many Pals as possible early on in order to
      level up quicker.
    * Bluey: The Video Game - I was curious to see what it was like and played
      the first level with my Fiancée and completed the rest of the game
      with her nephew. The game is not the best for people (well children)
      unfamiliar with platforms as it doesn't easy them into the typical
      mechanic of moving in a direction and jumping at the same time.
      Overall played 2 hours and collected 9 of the 22 achievements.
    * Call of Duty: Black Ops 6 - I'm glad I missed and didn't purchase the
      previous instalment between COD: Modern Warfare (the 2019 edition) as this
      one really hit the itch. The disappointing thing is they removed the My
      Call of Duty website and without the API for capturing matches that you
      played.
* Steam
    * This was sad year for my Steam's yearly review email came along and
      I discovered I had only played one game and only for one day.
* lichess
  * Played 350 games
  * played 12,553 moves
  * Spent 2 days and 8 hours playing
  * July saw me climb to my highest rating in Blitz.
  * December was very rough as I was on a loosing streak and sunk to almost to
    my lowest rating since the initial placement matches.

## Television

* Netflix
    * Black Lightning - Season 4 - Series Finale
    * Black Doves - Season 1
    * Noby Wants this - Season 1
    * Archer - Season 14 - Series Finale
    * Unstable - Season 2
    * My Demon - Season 1
    * Forecasting - Love & Weather - Season 1
    * Alice in Borderland - Season 2
    * The Brothers Sun - Season 4
    * THe Tree Body Problem - Season 1
* Disney+
  * Agatha All Along - Season 1
  * The Artful Doger - SEason 1
  * Echo - Season 1
  * Code Geass: Rozé of the Recapture - Season 1
  * The Acolyte - Season 1. \
    It is a shame this won't be getting as second season.
  * Doctor Who - First few eps with Ncuti Gatwa as The Doctor.
* Foxtel
    * The Penguin - Season 1
    * House of Dragon  - Season 2

The intention was to cancel Disney+ however I missed sorting it out by 3 days.

## Movies

* Howl's Moiving Castle
* Sprited Away
* Alita: Battle Angel
* Bevely HIlls Cop: Alex F
* Ford v Ferrari - Watched this on the plane
* Bohemian Rhapsody 
* Aquanman and the Lost Kingdom
* Blue Bettle
* America Ultra
* Rebel Moon - Part Two
* The Marvels
* Renfield
* Uncarted
* Everything Everywhere all at Once.
* Damsel
* Code 8 Part II
* Dune and Dune Part 2
* JUNG_E
* Carry-On
* Bullet Train

## Music

### Artists
* Charles Bardin (Soundtrack for Headbangers)
* Death Cab for Cutie
* Talyor Swift
* Say Hi
* Maya Hawke
* Muse

### Top Tracks
* Birds of a Feather - Billie Ellish
* Good Luck, Babe! - Chappell Roan
* I'm Just a Piegon - Charles Bardin
* Pigeon's Full Capacity - Charles Bardin
* Houdini - Dua Lipa

Many of the repeated tracks this year was due to listening to playlist for my
wedding.

### Things for 2025

No promises this time as I didn't do anything I said last year.

[0]: https://github.com/lucasg/Dependencies
[1]: https://github.com/lucasg
[2]: https://pdfium.googlesource.com/pdfium/
[3]: https://openapi-generator.tech/
[4]: https://ratatui.rs/
[5]: https://github.com/gabdube/native-windows-gui
[6]: https://ris.ripe.net/docs/mrt/
[7]: https://taiga.io/
[8]: https://plane.so/
[9]: https://github.com/hcengineering/huly-selfhost