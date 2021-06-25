# A Daskified yt : reading data

yt leverages Morton Indexing, a spatial bitmap-indexing algorithm, to efficiently map a regions of a physical domain to the data whether on-disk or elsewhere (cite yt4 paper, in prep). When a user then reads in data, in the case of an on-disk data store, only the datafiles that are known to contain data are opened, saving on computation. This approach naturally fits with a dask approach: rather than iterating over the datafiles, we aim to construct dask (or more correctly, dask-unyt arrays) in which the chunks of the array point to data determined by yt's spatial indexing.

```
def _read_particle_fields(self, fields, dobj, chunk=None):

        if len(fields) == 0:
            return {}, []
        fields_to_read, fields_to_generate = self._split_fields(fields)
        if len(fields_to_read) == 0:
            return {}, fields_to_generate


        # assemble the self.data_files that intersect the selector
        dfi, nfiles = self._identify_file_masks(dobj) 
        if nfiles == 0:
            return {}, fields_to_generate
        data_file_subset = [self.data_files[df_i] for df_i in dfi]
        
        # add on coordinates and smoothing_length to fields for every particle type if they're not there
        # so they will be available for our selector
        is_all_data = getattr(dobj.selector, "is_all_data", False)
        if is_all_data is False:
            for field in fields_to_read:
                # note: need to change Coordinates to particle_position....
                if (field[0], "Coordinates") not in fields_to_read:
                    fields_to_read.append((field[0], "Coordinates"))
                if (field[0], "smoothing_length") not in fields_to_read:
                    fields_to_read.append((field[0], "smoothing_length"))

        # read all the data from the intersecting data_files into delayed dask arrays
        fields_to_return = self.io._read_from_datafiles(data_file_subset, fields_to_read)
        
        # find the particles within the selector         
        if is_all_data is False:
            fields_to_return = self.apply_selector_mask(fields_to_return, dobj.selector)

        return fields_to_return, fields_to_generate 
```

## strategy: improving frontends 

One interesting consequence of this re-structiong is a simplification of yt's frontends. In yt, a frontend is a module that specifies how to properly read in a data format. Writing a new frontend involves, among other things, writing io routines. Currently in yt, this requires understanding not only how to read from the file format your frontend is designed for but also  yt's indexing and data selection methods. In order to make reading data as efficient as possible, after the indexing provides the datafiles to read, the data selection criteria are then applied to each datafile and then concatenated. In the case of the GadgetHDF5 frontend, this looks like:

```
def _read_particle_fields(self, chunks, ptf, selector):
        # Now we have all the sizes, and we can allocate
        data_files = set()
        for chunk in chunks:
            for obj in chunk.objs:
                data_files.update(obj.data_files)
        for data_file in sorted(data_files, key=lambda x: (x.filename, x.start)):
            si, ei = data_file.start, data_file.end
            f = h5py.File(data_file.filename, mode="r")
            for ptype, field_list in sorted(ptf.items()):
                if data_file.total_particles[ptype] == 0:
                    continue
                g = f[f"/{ptype}"]
                if getattr(selector, "is_all_data", False):
                    mask = slice(None, None, None)
                    mask_sum = data_file.total_particles[ptype]
                    hsmls = None
                else:
                    coords = g["Coordinates"][si:ei].astype("float64")
                    if ptype == "PartType0":
                        hsmls = self._get_smoothing_length(
                            data_file, g["Coordinates"].dtype, g["Coordinates"].shape
                        ).astype("float64")
                    else:
                        hsmls = 0.0
                    mask = selector.select_points(
                        coords[:, 0], coords[:, 1], coords[:, 2], hsmls
                    )
                    if mask is not None:
                        mask_sum = mask.sum()
                    del coords
                if mask is None:
                    continue
                for field in field_list:

                    if field in ("Mass", "Masses") and ptype not in self.var_mass:
                        data = np.empty(mask_sum, dtype="float64")
                        ind = self._known_ptypes.index(ptype)
                        data[:] = self.ds["Massarr"][ind]
                    elif field in self._element_names:
                        rfield = "ElementAbundance/" + field
                        data = g[rfield][si:ei][mask, ...]
                    elif field.startswith("Metallicity_"):
                        col = int(field.rsplit("_", 1)[-1])
                        data = g["Metallicity"][si:ei, col][mask]
                    elif field.startswith("GFM_Metals_"):
                        col = int(field.rsplit("_", 1)[-1])
                        data = g["GFM_Metals"][si:ei, col][mask]
                    elif field.startswith("Chemistry_"):
                        col = int(field.rsplit("_", 1)[-1])
                        data = g["ChemistryAbundances"][si:ei, col][mask]
                    elif field == "smoothing_length":
                        # This is for frontends which do not store
                        # the smoothing length on-disk, so we do not
                        # attempt to read them, but instead assume
                        # that they are calculated in _get_smoothing_length.
                        if hsmls is None:
                            hsmls = self._get_smoothing_length(
                                data_file,
                                g["Coordinates"].dtype,
                                g["Coordinates"].shape,
                            ).astype("float64")
                        data = hsmls[mask]
                    else:
                        data = g[field][si:ei][mask, ...]

                    yield (ptype, field), data
            f.close()

```

but by using dask arrays, we can read in each of those datafiles chunks in a delayed fashion and apply the selector objects later, which means that a frontend need only specify how to pull data from a single datafile:

```python

def _read_particle_fields(self, data_file, ptf):

        si, ei = data_file.start, data_file.end
        f = h5py.File(data_file.filename, mode="r")
        for ptype, field_list in sorted(ptf.items()):
            if data_file.total_particles[ptype] == 0:
                continue
            g = f[f"/{ptype}"]                            
            for field in field_list:

                if field in ("Mass", "Masses") and ptype not in self.var_mass:
                    data = np.empty(mask_sum, dtype="float64")
                    ind = self._known_ptypes.index(ptype)
                    data[:] = self.ds["Massarr"][ind]
                elif field in self._element_names:
                    rfield = "ElementAbundance/" + field
                    data = g[rfield][si:ei][mask, ...]
                elif field.startswith("Metallicity_"):
                    col = int(field.rsplit("_", 1)[-1])
                    data = g["Metallicity"][si:ei, col][mask]
                elif field.startswith("GFM_Metals_"):
                    col = int(field.rsplit("_", 1)[-1])
                    data = g["GFM_Metals"][si:ei, col][mask]
                elif field.startswith("Chemistry_"):
                    col = int(field.rsplit("_", 1)[-1])
                    data = g["ChemistryAbundances"][si:ei, col][mask]
                elif field == "smoothing_length":
                    # This is for frontends which do not store
                    # the smoothing length on-disk, so we do not
                    # attempt to read them, but instead assume
                    # that they are calculated in _get_smoothing_length.
                    if hsmls is None:
                        hsmls = self._get_smoothing_length(
                            data_file,
                            g["Coordinates"].dtype,
                            g["Coordinates"].shape,
                        ).astype("float64")
                    data = hsmls[mask]
                else:
                    data = g[field][si:ei][mask, ...]

                yield (ptype, field), data
        f.close()

```

## examples 

show some examples 

## performance

show initial performance comparisons

