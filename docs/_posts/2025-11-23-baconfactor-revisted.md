---
layout: post
title:  "Revisting baconfactor"
date:   2025-11-23 18:00:00 +0930
---

Last weekend, I was catching up with friends and the game of "Six Degrees of
Kevin Bacon" was put forward but with a focus on television shows. The two shows
chosen were Sliders and Last of Us. Afterwards, it got me to thinking more about
what shows I remember seeing certain people in and that I was keen to dust off
the program I hade made to to do it called "baconfactor".

Due to the data I had for cast and filmography being several years out of date
it lacked **Last of Us**. To account for that instead I had to change the
destination to a TV show that it did know about. For that the show I knew about
was Agents of S.H.I.E.L.D, which has Ming-Na Wen who plays Melinda May who plays
Fennec Shand in The Mandalorian, which has Pedro Pascal as the title character
who I knew was in Last of Us (didn't know his character name).

The result it gave was:
* Agents of S.H.I.E.L.D. (2364582)
* Titus Welliver (920038)
* CSI: Crime Scene Investigation (247082)
* Fredric Lehne (499791)
* Sliders (112167)

The numbers next to the names are the IMDB IDs of the people (names) and
shows (titles). I wasn't keen on it giving CSI series as an answer as I'm really
not familiar with that and there are too many people it. That got me thinking
maybe if the actors were weighted based on how many episodes they were in so
it would favour the main cast and recurring people over guest stars that would
be improvement. Personally, I find that I often it is memorable guest stars that
make finding the connection easier to remember. Possibly another improvement
would be to be able to provide it a list of shows that someone has seen and
actors they familiar with so it could favour them.

## File Formats

There are four files that the program uses as its database.

* `actors.dat` - List of IMDB IDs and the names of the actors.
* `titles.dat` - List of IMDB IDs and the names of the shows.
* `actors_to_titles.dat` - List of IMDB IDs of the actors and then for each
  actor a list of the shows they are in.
* `titles_to_actors.dat` - List of IMDB IDs of the titles and then for each
  actor a list of the shows they are in.

As you see it was very IMDB heavy, after all the first version of the program
was called `imdbLinker` rather than `baconfactor`. The newer program simply
took the IMDB ID of the shows, but offered a basic search feature to find a
show's ID by giving it part of the name and it would return several results.

The first two files are the same format, and the last two files are same format
as one another.

### ID and Name

The file format for the IDs and names used are:

- ID Count - u32
- ID[0]    - u32
- ....     - u32 (each)
- ID[ID Count - 1] - u32
- NameLength[0] - u8
- Name[0]       - char[NameLength[0]]
- NameLength[1] - u8
- Name[1]       - char[NameLength[1]]
- ..
- NameLength[ID Count - 1] - u8
- Name[ID Count - 1]       - char[NameLength[ID Count - 1]]

This means there is firstly the number of how many IDs there are (the ID count),
followed by each of the IDs. After that is a byte that length of the name
(i.e. how many chars/bytes) it makes up, which is then followed by that many
characters.

### Index file
This define an index for looking up the associate between actors and shows or
shows and actors respectively.

- Four byte header:
  * Magic number: 0xbf (bactorfactor)
  * File version: 0x01 - Version 1
  * File features: 1 byte (0 for links are IDs and 1 for links are indices)
  * Reserved     : 1 byte - ignored.
- Source Count : u32
- Source ID 0
- ...
- Destination Count : u32  (this is at (Source Count) * 4 bytes) after..
- Offset for destination of first source.
- Offset for destination of second source.
- ...
- Destination count for Source[0]
- Destination[0] for Source[0]
- Destination[1] for Source[0]
- Destination[Count0 - 1] for Source[0]
- Destination count for Source[1]
- Destination[0] for Source[1]
- Destination[1] for Source[1]
- Destination[Count1 - 1] for Source[1]

The reason it was this way rather than [Offset, Count], [Offset, Count]
is lost to time and my memory. The other approach I've seen used by Boost (the
widely used C++ library) is to have the Count be implicit, where you know if
you only have the offsets  [OffsetToA, OffsetToB,...] that `OffsetToA` ends at
`OffsetToB` so the count is implicit, you simply need one extra offset more
to represent the end of the last source.

The source IDs are included so if you want to purely work between files however
since they in the other file and for the output to be useful it needs to
converted to names that is used instead. The expectation is the order of IDs
in their respective files are the same. This means `Source[i] = 112167` and
`ID[i] = 112167` where `i` is the same in both.

To find what actors appear in a particular show, for example "The Following" then
first, would find its ID (`112167`) in list of Source IDs, which would give us
the index into Destination of that particular show. Lets say it index `2` then
we look-up `Destination[2]` which gives the offset into the Destination array.
which we look-up the count, lets just say the count was 10. From, from there
the next 10 numbers are read. What the numbers represent depend on the
`file features`.

As mentioned the format was worked heavily around IMDB IDs so the first version
of the file format had Destination refer to IDs. In this case the numbers at
destination represent the IMDB IDs of the actors in that show.

This however slowed things down as it meant once you found an actor, you then
had to find the index of their ID to then find their shows. To avoid, that the
format was changed to instead use the indices in the Destination. This way
it could jump straight to the Destination array of the actor without doing the
look-up of the ID.

## Features

This is basically to give some background about what the program could do.
It may removed from this post one day and move it to a dedicated project
page.

### Listings

The listing commands were useful to check what it was in the database.
Perform the seach command was added it also provided way to search for name
by using `grep` or `findstr` (Windows) on the output.
s
* List the IDs and names of shows.
* List the IDs and names of actors
* List filmography of an actor.
* List the cast of a show.

For example, the listing of the cast of the show "The Following" is:
````
Enter ID of show: 2071645
Actors in The Following(2071645) appearing:
There are 169 actors in the show.
102     Kevin Bacon
39162   Shawn Ashmore
700856  James Purefoy
3862996 Sam Underwood
1888211 Jessica Stroup
2038170 Valorie Curry
954036  Natalie Zea
733196  Zuleikha Robinson
...
```

### Name Search
Ability to search for the IDs of a show or actor by a string. This was very
basic case-insensitive match where it simply had to have the string given
anywhere in the name.

The following shows a search for `lost`.
```
Enter search term: lost
Search results:
69638   The Starlost
86712   Finder of Lost Loves
240278  The Lost World
411008  Lost
830361  The Lost Room
1150692 Lost: Missing Pieces
1300170 Xam'd: Lost Memories
1314008 Lost Tapes
1429449 Lost Girl
3235906 Lost and Found: Loster
3654206 Defiance: The Lost Ones
```

## New Version

I wanted to be able to easily share this with the friends without them needing
to download anything so I set off porting the the program to JavaScript so it
could be hosted on a web page and shared.

I originally had started a JavaScript implementations using Node.Js but didn't
get very far. The most I had was a few lines for doing the initial file read
and maybe reading the first integer out. I ended up switching to Deno instead of
Node.Js.

### Reading data

Started with reading the list of IDs and names which ended up being:
```javascript
// Read file containing the list of IDs and name.
function ReadList(buffer)
{
  var count = (new Uint32Array(buffer, 0, 8))[0];
  let ids = new Uint32Array(buffer, 0, count);
  var actorCount = ids[0];
  var data = {
    'count': ids[0],
    'ids': ids.slice(1, ids[0]),
  };

  // Read in the names.
  const decoder = new TextDecoder();
  let rawNames = new Uint8Array(buffer, 4 + actorCount * 4);
  var names = [];
  for (var index = 0, i = 0; i < actorCount; ++i, ++index)
  {
    var length = rawNames[index];
    const name = rawNames.slice(index + 1, index + 1 + length);
    const str = decoder.decode(name);
    names.push(str);
    index += length;
  }

  data['names'] = names;
  return data;
}
```

Next up was the index file.

```javascript
class Index {
  constructor(sources, links) {
    this.sources = sources;
    this.links = links;
  }

  // Create an Index object from a buffer.
  //
  // This can read the index of mapping actors to shows or shows to actors.
  static fromBuffer(buffer) {
    const data = new Uint32Array(buffer);
    const header = data[0];
    // The header is ignored for now - and assume feature is links are indices.
    const sourceCount = data[1];
    const targetCount = data[1 + sourceCount];
    return new Index(data.slice(2, 2 + sourceCount),
                     data.slice(3 + sourceCount));
  }

  // Return the index of the given source given their ID.
  indexById(id)
  {
    return this.sources.indexOf(id);
  }

  // Return the connection for the source at the given index.
  //
  // The connections are the indices rather than IDs.
  connections(index) {
    const connectionOffset = this.links[index];
    const connectionCount = this.links[connectionOffset];
    return this.links.slice(connectionOffset + 1,
                            connectionOffset + connectionCount + 1);
  }
}
```

This was a bit suprising as orignally it was looking quite messy but once things
started falling in to place it worked out.

### Computing sequence

Normally the first thing it does is 

```javascript
// showsToActors and actorsToShows are the respective Index above.
function findSequence(startShow, endShow, showsToActors, actorsToShows)
{
  const startShowIndex = showsToActors.indexById(startShowId);
  const endShowIndex = showsToActors.indexById(endShowId);
  var visitedShowIndices = new Set();

  var queue = [{"sequence": [startShowIndex]}];
  while (queue.length)
  {
    var node = queue.shift();
    const actors = showsToActors.connections(node.sequence[node.sequence.length - 1]);
    node.sequence.push(0); // Place holder for the next actor in the sequence.

    // For each actor that appeared in the show,
    // find what shows they were in.
    const keepGoing = actors.every((actorIndex) => {
      const shows = actorsToShows.connections(actorIndex);

      // Update the actor index in the sequence.
      node.sequence[node.sequence.length - 1] = actorIndex;

      // Check to find out if we are the end.
      if (shows.find((show) => show == endShowIndex))
      {
        node.sequence.push(endShowIndex);
        return false;
      }

      // Now each show the actor was in needs to be visited if we haven't
      // didn't find it.
      shows.forEach((showIndex) => {
        if (!visitedShowIndices.has(showIndex))
        {
          // Add the new show to the queue.
          let newSequence = node.sequence.slice();
          newSequence.push(showIndex);
          queue.push({"sequence": newSequence});

          // Mark it as visited now so if another actori s also in the show it
          // is ignored.
          visitedShowIndices.add(showIndex);
        }
      });
            
      return true;
    });

    // If one of the actors was in the end show, then we don't keep going and 
    // we have found the a sequence between the shows.
    if (!keepGoing)
    {
      return node.sequence;
    }
  }
  return [];
}
```

This gives only the indices, I then have a function for going to the names.
```javascript
function resolveSequence(sequence, showList, actorList)
{
  return sequence.map((element, index) => {
    if (index & 1)
    {  
      const name = actorList["names"][element];
      const id = actorList["ids"][element];
      return `${name} (${id})`
    }
    const name = showList["names"][element];
    const id = showList["ids"][element];
    return `${name} (${id})`
  })
}
```
### Implementations reflection
That pretty much all there is too it and is it. The actual version had another
class to tie the data together.

Since the plan was to host a website with it, it seemed like a better idea to
skip the name reading from the binary file at runtime and convert it to a
JavaScript file ahead of time. After all in this case in JavaScript each
name would start and end with a " " and have comma after it (except last) so
that means 3 bytes per name for the JavaScript syntax. As the IDs are already
in the index, that saves 4 bytes per name and there is also the extra byte 
for its length that isn't needed so this comes out 2 bytes less for each
actor or show.

Example of the output of the `names.generated.js` is:
```javascript
export const actorNames = ["Kevin Bacon","Bo Derek","Skeet Ulrich", ...
export const titleNames = ["Kraft Theatre","Studio One in Hollywood", ...
```

Very much happy with how it turned out. The hardest part was fighting the
various tools for reading the data in JavaScript. A lot of trail and error
was carried out to go from reading a file in Deno (and Node before it) to
being able to read 32-bit integers or bytes and strings out.

## Conclusion

The main focus here was on how it works and thus the file format and
implementation. I was very happy with the result. It was very tempting to try
to port it to Rust but I really did not need another distraction. The next
part of the project was the web page aspect, the basics for selecting the
start and end show have been done and hooking up the `findSequence()` and
`resolveSequence()` functions has been done but I'm not happy with the output
(styling and layout).

I started looking at updating the data but its more difficult to collect the
data from IMDB and it needed reworking anyway as the script was written for
Python 2. Going forward the data itself would most likely come from
[TMDB](https://www.themoviedb.org/) instead of IMDB which would likely see me
drop the IMDB ID usage as getting the mapping from TMDB to IMDB is not straight
forward.

