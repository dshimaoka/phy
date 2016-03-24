# Manual clustering

The `phy.cluster.manual` package provides manual clustering routines. The components can be used independently in a modular way.

## Clustering

The `Clustering` class implements the logic of assigning clusters to spikes as a succession of undoable merge and split operations. Also, it provides efficient methods to retrieve the set of spikes belonging to one or several clusters.

Create an instance with `clustering = Clustering(spike_clusters)` where `spike_clusters` is an `n_spikes`-long array containing the cluster number of every spike.

Notable properties are:

* `clustering.spikes_per_cluster`: a dictionary `{cluster_id: spike_ids}`.
* `clustering.cluster_ids`: array of all non-empty clusters
* `clustering.spike_counts`: dictionary with the number of spikes in each cluster

Notable methods are:

* `clustering.new_cluster_id()`: generate a new unique cluster id
* `clustering.spikes_in_clusters(cluster_ids)`: return the array of spike ids belonging to a set of clusters.
* `clustering.merge(cluster_ids)`: merge some clusters.
* `clustering.split(spike_ids)`: create a new cluster from a set of spikes. **Note**: this will change the cluster ids of all affected clusters. For example, splitting a single spike belonging to cluster 10 containing 100 spikes leads to the deletion of cluster 10, and the creation of clusters N (99 spikes) and N+1 (1 spike). This is to ensure that cluster ids are always unique.
* `clustering.undo()`: undo the last operation.
* `clustering.redo()`: redo the last operation.

### UpdateInfo

Every clustering action returns an `UpdateInfo` object and emits a `cluster` event.

An `UpdateInfo` object is a `Bunch` instance (dictionary with dotted attribute access) with several keys, including:

* `description`: can be `merge` or `assign`
* `history`: can be `None` (default), `'undo'`, or `'redo'`
* `added`: list of new clusters
* `deleted`: list of removed clusters

A `Clustering` object emits the `cluster` event after every clustering action (including undo and redo). To register a callback function to this event, use the `connect()` method.

Here is a complete example:

```python
>>> import numpy as np
>>> from phy.cluster.manual import Clustering
```

```python
>>> clustering = Clustering(np.arange(5))
```

```python
>>> @clustering.connect
... def on_cluster(up):
...     print("A %s just occurred." % up.description)
```

```python
>>> up = clustering.merge([0, 1, 2])
A merge just occurred.
```

```python
>>> clustering.cluster_ids
array([3, 4, 5])
```

```python
>>> clustering.spike_counts
{3: 1, 4: 1, 5: 3}
```

```python
>>> for key, val in up.items():
...     print(key, "=", val)
undo_state = None
description = merge
metadata_value = None
spike_ids = [0 1 2]
old_spikes_per_cluster = {0: array([0]), 1: array([1]), 2: array([2])}
descendants = [(0, 5), (1, 5), (2, 5)]
added = [5]
new_spikes_per_cluster = {5: array([0, 1, 2])}
history = None
metadata_changed = []
deleted = [0, 1, 2]
```

## Cluster metadata

The `ClusterMeta` class implement the logic of assigning metadata to every cluster (for example, a cluster group) as a succession of undoable operations.

Here is an example.

```python
>>> from phy.cluster.manual import ClusterMeta
>>> cm = ClusterMeta()
```

```python
>>> cm.add_field('group', default_value='unsorted')
```

```python
>>> cm.get('group', 3)
'unsorted'
```

```python
>>> cm.set('group', 3, 'good')
<metadata_group [3] => good>
```

```python
>>> cm.set('group', 3, 'bad')
<metadata_group [3] => bad>
```

```python
>>> cm.get('group', 3)
'bad'
```

```python
>>> cm.undo()
<metadata_group (undo) [3] => bad>
```

```python
>>> cm.get('group', 3)
'good'
```

You can import and export data from a dictionary using the `to_dict()` and `from_dict()` methods.

```python
>>> cm.to_dict('group')
{3: 'good'}
```

## Views

There are several views typically associated with manual clustering operations.

### Waveform view

The waveform view displays action potentials across all channels, following the probe geometry.

### Feature view

The feature view shows the principal components of spikes across multiple dimensions.

### Trace view

The trace view shows the continuous traces from multiple channels with spikes superimposed. The spikes are in white except those belonging to the selected clusters, which are in the colors of the clusters.

### Correlogram view

The correlogram view computes and shows all pairwise correlograms of a set of clusters.

## Manual clustering GUI component

The `ManualClustering` component encapsulates all the logic for a manual clustering GUI:

* cluster views
* selection of clusters
* navigation with a wizard
* clustering actions: merge, split, undo stack
* moving clusters to groups

Create an object with `mc = ManualClustering(spike_clusters)`. Then you can attach it to a GUI to bring manual clustering facilities to the GUI: `mc.attach(gui)`. This adds the manual clustering actions and the two tables to the GUI: the cluster view and the similarity view.

The main objects are the following:

`mc.clustering`: a `Clustering` instance
`mc.cluster_meta`: a `ClusterMeta` instance
`mc.cluster_view`: the cluster view (derives from `Table`)
`mc.similarity_view`: the similarity view (derives from `Table`)
`mc.actions`: the clustering actions (instance of `Actions`)

Use `gui.request('manual_clustering')` to get the `ManualClustering` instance inside the `attach_to_gui(gui, model=None, state=None)` method of a GUI plugin.

### Cluster and similarity view

The cluster view shows the list of all clusters with their ids, while the similarity view shows the list of all clusters sorted by decreasing similarity wrt the currently-selected clusters in the cluster view.

You can add a new column in both views as follows:

```python
>>> @mc.add_column
... def n_spikes(cluster_id):
...     return mc.clustering.spike_counts[cluster_id]
```

The similarity view has an additional column compared to the cluster view: `similarity` with respect to the currently-selected clusters in the cluster view.

See also the following methods:

* `mc.set_default_sort(name)`: set a column as default sort in the quality cluster view
* `mc.set_similarity_func(func)`: set a similarity function for the similarity view

### Cluster selection

The `ManualClustering` instance is responsible for the selection of the clusters.

* `mc.select(cluster_ids)`: select some clusters
* `mc.selected`: list of currently-selected clusters

When the selection changes, the attached GUI raises the `select(cluster_ids, spike_ids)` event.

Other events are `cluster(up)` when a clustering action occurs, and `request_save(spike_clusters, cluster_groups)` when the user wants to save the results of the manual clustering session.

## Cluster store

The **cluster store** contains a library of functions computing data and statistics for every cluster. These functions are cached on disk and possibly in memory. A `ClusterStore` instance is initialized with a `Context` which provides the caching facilities. You can add new functions with the `add(f)` method/decorator.

The `create_cluster_store()` function creates a cluster store with a built-in library of functions that take the data from a model (for example, the `KwikModel` that works with the Kwik format).

Use `gui.request('cluster_store')` to get the cluster store instance inside the `attach_to_gui(gui, model=None, state=None)` method of a GUI plugin.

## GUI plugins

You can create plugins to customize the manual clustering GUI. Here is a complete example showing how to change the quality and similarity measures. Put the following in `~/.phy/phy_config.py`:

```python

import numpy as np
from phy import IPlugin
from phy.cluster.manual import get_closest_clusters


# We write the plugin directly in the config file here, for simplicity.
# When dealing with more plugins it is a better practice to put them
# in separate files in ~/.phy/plugins/ or in your own repo that you can
# refer to in c.Plugins.dirs = ['/path/to/myplugindir'].
class MyPlugin(IPlugin):
    def attach_to_gui(self, gui, model=None, state=None):

        # We can get GUI components with `gui.request(name)`.
        # These are the two main components. There is also `context` to
        # deal with the cache and parallel computing context.
        mc = gui.request('manual_clustering')
        cs = gui.request('cluster_store')

        # We add a column in the cluster view and set it as the default.
        @mc.add_column(default=True)
        @cs.add(cache='memory')
        def mymeasure(cluster_id):
            # This function takes a cluster id as input and returns a scalar.

            # We retrieve the spike_ids and waveforms for that cluster.
            # spike_ids is a (n_spikes,) array.
            # waveforms is a (n_spikes, n_samples, n_channels) array.
            spike_ids, waveforms = cs.waveforms(cluster_id)
            return waveforms.max()

        def mysim(cluster_0, cluster_1):
            # This function returns a score for every pair of clusters.

            # Here we compute a distance between the mean masks.
            m0 = cs.mean_masks(cluster_0)  # (n_channels,) array
            m1 = cs.mean_masks(cluster_1)  # (n_channels,) array
            distance = np.sum((m1 - m0) ** 2)

            # We need to convert the distance to a score: higher = better
            # similarity.
            score = -distance
            return score

        # We set the similarity function.
        @mc.set_similarity_func
        @cs.add(cache='memory')
        def myclosest(cluster_id):
            """This function returns the list of closest clusters as
            a list of `(cluster, sim)` pairs.

            By default, the 20 closest clusters are kept.

            """
            return get_closest_clusters(cluster_id, model.cluster_ids, mysim)


# Now we set the config object.
c = get_config()

# Here we say that we always want to load our plugin in the KwikGUI.
c.KwikGUI.plugins = ['MyPlugin']

```