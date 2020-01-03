
### Other SCC (Perforce)

[Perforce](https://www.perforce.com/) is fairly common in the game industry.  Should you use that, [p4 streams](https://www.perforce.com/perforce/r13.1/manuals/p4v/streams_overview.html) enable a workflow where Lumberyard releases can be treated as external code drops and go through integration/stabilization before getting merged to mainline/trunk.

Here's an example from one of our projects:  
![](/assets/p4_streams_lmbr.png)

| Name | Type | Description
|-|-|-|
| upstream | release | Lumberyard releases
| wiplmbrd | release | Integration branch
| patches | development | Hot fixes to Lumberyard (between releases)
| main
