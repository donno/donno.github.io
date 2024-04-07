---
layout: post
title:  "DLL dependency graphs"
date:   2024-03-24 20:00:00 +0930
---

Producing graphs of the dependencies between Window DLLs (dynamic link
libraries). The goal is to use this information to make it easy to see if a DLL
fails to load if it is likely due to its dependency by showing what it depends
on.

## Motivation
The motivation for this project was that I was looking into trying to run
various Windows programs on Linux with Wine. I started out with a simple Python
script which would load the various DLLs. This would let me know if a
which DLLs couldn't be loaded. The problem was it didn't tell me why they
failed and the most common reason was because one of their dependencies (the
DLLs that it loads) couldn't be loaded. The solution I decided to go with to
this problem is to build a graph of the DLLs and their dependencies so you
could visually see which libraries were causing the most issues.

## Process

1. Downloads and extracts the [Dependencies][1] tool by [Lucasg][2]
2. Produce JSON output using the `-json` command line argument and the 
   `-chain` with `-depth 2`. The depth argument limits it taking a long time.
3. Cache the resulting JSON. This speeds up the process on subsequent runs.
4. Convert the JSON output to a simplified graph structure where it keeps 
   just enough information to have the nodes and edges themselves.
5. Download and extract [GraphViz][4].
6. Uses GraphViz to perform transitive reduction using the `tred` program.
   This removes the edges between lower-level libraries if there is already a 
   path through it via another library it depends on. Ultimately, this reduces
   clutter.
7. Uses GraphViz to create a PNG as well as DOT version with the layout
   performed (i.e assign positions to the nodes).
8. Outputs to Gephi's JSON format, as the intention was to use it with the
   Network component of [vis.js][3]. However, that library can also do DOT. 

The resulting script is [dll_dependencies.py][0].

## Example
![Graph of GraphVIz DLLs without transient dependencies removed](/assets/2024-03-24-dll_depend_graph_example.png)

![Graph of GraphVIz DLLs with transient dependencies removed](/assets/2024-03-24-dll_depend_graph_example_post_tred.png)

## Wine Project

The DOT graph is then consumed by the script for the Wine project which
colours the nodes based on if the library could be loaded or not.

The next time I pick-up this project, I might look at reworking it to make it
available to others and write a follow-up post for it.

## Future

* Rework `find_dependencies()` to use process multiple DLLs in parnell.
* Make the DLL load project public as well. It not limited to Wine as it can
  also be used for the different Window containers base images.

[0]: https://github.com/donno/warehouse51/blob/master/devtools/dll_dependencies.py
[1]: https://github.com/lucasg/Dependencies
[2]: https://github.com/lucasg
[3]: https://visjs.org/
[4]: https://graphviz.org/