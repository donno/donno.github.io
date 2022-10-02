---
layout: page
---

This are mostly software projects that I worked on over the years.

## baconfactor
A project that uses the idea of "Six Degrees of Kevin Bacon", but with a focus
on television shows. At the time, I wasn't aware that that was the name
given to it.

The original project was written around June and July in 2007 where it had the
name imdbLinker due to IMDB being the primary data source.

Around 2017, with a renewed interest it I wrote a newer version based on the
database I still had. This new version used a different approach and was far
more optimal. The key benefit was the data preparation and use of a
struct of arrays rather than a SQLite database for the queries.

## descentreader
A tool/library for reading the level file formats of the video game by
Parallax Software called "Descent".

Descent is a 3D six degrees of freedom shooter where the player is in a ship
and has to fly through mine tunnels of a labyrinthine nature and destroy
virus infected robots.

## relaxtolife
Produces a calender that shows how I spend my time relaxing. This includes
Call of Duty matches, Television/Movies from Trakt and sleep from FitBit.

It could also include PUBG as I have information about almost every game I
played.

This project is in what I am calling the incubating state and is currently
private.

## DLX tools

DLX is a RISC processor architecture designed by John L. Hennessy and David A.
Patterson. The university I ended in 2007 to 2010 used a slightly modified
version of this processor for teaching computer systems. After I left they
moved to RISC-V.

### DLX MIDI Device

I developed a plugin for the University's DLX simulator (written in Java) that
allowed you to write a program that could play music by writing the MIDI codes
to a memory address (the idea was the system used memory mapped-IO).

### NDLX - .NET DLX

A framework for developing algorithms and applications for DLX using .NET
Framework. This was specifically aimed at C#.
Essentially how it worked is I would write the algorithm in C# normally and
and write tests for it. Next, I would rewrite it without while and for by
using if and goto (a.k.a 'jump'). From there, it would be a matter of
register allocation and changing the code to use the functions that
represent, so result += 8 would be changed to addui(r1, r1, 8);

The idea is that would simulate the program and rerunning the tests would
confirm that I used the right registers and mapped it to the right
instructions. This when hand-in-hand with some rules I had of how to
convert the C# code to assembly.

## gitweb
[GitHub Project](https://github.com/donno/gitweb)

A project a project written in C++ that provides the data (commits and blobs)
from Git repositories as JSON inspired by the GitHub V3 APIs..

This used libgit2 to interact with a Git repository.

I built a simple website using the
[Atlassian User Interface](https://aui.atlassian.com/) framework.

**API Example**
| URI           | Description   |
| ------------- |:-------------:|
| /api/repos/{repo-name} | Summary of that repo. |
| /api/repos/{repo-name}/branches | List the branches in that repo |
| /api/repos/{repo-name}/tags | List the tags in that repo |
| /api/repos/{repo-name}/tags/{name} | Information about that tag. |
| /api/repos/{repo-name}/commit/{hash} | Information for that hash.|

## Space Invaders for XBMC

This project was started around April/May 2006.

A implementation of the classic arcade game known as Space Invaders for the
XBMC (the media center). As a reminder, XBMC become Kodi.

Features
* Background Music - Optional 'Overwrite music' with Custom Playlist.pls in
  Music folder
* Internet and Local Highscores
* Red Bonus Saucer
* Themes

Around 2017, I rewrote the the basics of the game for PyGame without the
menu system as that heavily required on XBMC to do the heavy lifting.
The source for this rewrite is available in my main GitHub repository.
[Source on GitHub](https://github.com/donno/warehouse51/tree/master/vaders)

## Others
Hopefully at some point I will break down each of the following further.

- breakout - Python - An implementation of the old classic game, Breakout.
- digital - Python - Python modules for representing digital electronics.
- numbertarget - Lua - A implementation of the Numbers game from the British TV
  program "Countdown" written in Lua using [Love 2D](https://love2d.org/).
- numberhunt - Python - An attempt at a solver for the Numbers game in
  the British TV program "Countdown"
- wordmatch - C++ - Matches a sequence of letters to a word.
  This is inspired to be a solver for Countdown.
  I re-purposed some of this work to help me play Wordle prior the that game
  being sold to the New York Times.
- peparser - Python/JavaScript - Parses the PE/COFF format commonly used for
  executables.
- hgt - Python & C++ - Provides a module for working with HGT file format which
  is the data file of the Shuttle Radar Topography Mission (SRTM).
- pdbtools - Python & C++ - Tools for working with PDB (Program Database files)
- sweeper - Python - An implementation of the old classic game, Minesweeper.

## Not Quite Started Projects

These are mostly projects that never really got very far.

* GW-BASIC Interpreter
* Spotify Scrobbler for Last.FM. Spotify already provided integration to do
  this without any extra work from my side.
* Chrome Dino for Atari 800\
  Implement the Dino game from Chrome (chrome://dino) for the Atari 800 or
  similar generation hardware.
* MapTODO\
  A small multi-user (small group) TODO list where the things to do are
  visiting places on a map.
