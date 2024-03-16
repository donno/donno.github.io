
Write-up about the script at:
https://github.com/donno/warehouse51/blob/master/devtools/dll_dependencies.py


Outline/notes
* Uses the [Dependencies][0] tool by [Lucasg][1]
* Converts the JSON output of the -chain command to a graph.
* Uses GraphViz to perform transitive reduction
* Uses GraphViz to create a PNG.
* Outputs to Gephi's JSON format, as the intention was to use it with the
  Network component of [vis.js][2].

[0]: https://github.com/lucasg/Dependencies
[1]: https://github.com/lucasg
[2]: https://visjs.org/