---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.12.0
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

<p><font size="6"><b>Xarray Datasets</b></font></p>


> *DS Python for GIS and Geoscience*  
> *October, 2020*
>
> *© 2020, Joris Van den Bossche and Stijn Van Hoey. Licensed under [CC BY 4.0 Creative Commons](http://creativecommons.org/licenses/by/4.0/)*

---

```{code-cell} ipython3
import numpy as np
import matplotlib.pyplot as plt

import xarray as xr
```

## Meet `xarray.Dataset`

+++

We already know `xarray.DataArray`: a single array with dimensions (and each dimension can have a name and coordinate values):

```{code-cell} ipython3
xr_array = xr.open_rasterio("./data/gent/raster/2020-09-17_Sentinel_2_L1C_B0408.tiff")
xr_array
```

In addition, xarray also has a `Dataset` object to store *multiple* arrays that have shared dimensions (but not all dimensions need to be shared).

For the initial example, let's convert our DataArray from above into a Dataset:

```{code-cell} ipython3
xr_array = xr_array.assign_coords(band=("band", ["b4", "b8"]))
xr_data = xr_array.to_dataset(dim="band")
```

```{code-cell} ipython3
xr_data
```

Let's take a closer look at this `xarray.Dataset`:

- A `xarray.Dataset` is the second main data type provided by xarray
- This example has 2 __dimensions__:
    - `y`: the y coordinates of the data set
    - `x`: the x coordinates of the data set
- Each of these dimensions are defined by a __coordinate__ (1D) array
- It has 2 __Data variables__: `b4` and `b8` that share the same coordinates
- Other metadata provided by the `tiff` are stored in the __`Attributes`__

Each of the data variables can be accessed as a single DataArray:

```{code-cell} ipython3
type(xr_data["b4"])
```

Why do we want to use a Dataset instead of DataArray?

* We can actually have multiple arrays that are logically different variables, eg temperature and pressure with the same (lon, lat, time) dimensions
* For some cases it can be more convenient to work with

+++

Recap the NDVI exercise of the previous notebook, using a stacked version of the 4th and 8th Sentinel band:

```{code-cell} ipython3
xr_array = xr.open_rasterio("./data/gent/raster/2020-09-17_Sentinel_2_L1C_B0408.tiff")
xr_array
```

In Numpy, we would do:

```{code-cell} ipython3
b48_bands = xr_array.values  # 0 is band 4 and 1 is band 8
b48_bands.shape
```

```{code-cell} ipython3
ndvi_np = (b48_bands[1] - b48_bands[0])/(b48_bands[0] + b48_bands[1]) # or was it b48_bands[0] -  b48_bands[1] ?
```

```{code-cell} ipython3
plt.imshow(ndvi_np, cmap="YlGn")
```

In __xarray__:

```{code-cell} ipython3
xr_array = xr_array.assign_coords(band=("band", ["b4", "b8"]))
xr_data = xr_array.to_dataset(dim="band")
```

```{code-cell} ipython3
ndvi_xr = (xr_data["b8"] - xr_data["b4"])/(xr_data["b8"] + xr_data["b4"])
```

```{code-cell} ipython3
plt.imshow(ndvi_xr, cmap="YlGn")
```

The result is the same, but no more struggling on what index is representing which variable!

```{code-cell} ipython3
np.allclose(ndvi_xr.data, ndvi_np)
```

We can keep the result together with the other data variables by adding a new variable to the data, in a very similar way as we created a new column in Pandas:

```{code-cell} ipython3
xr_data["ndvi"] = ndvi_xr
xr_data
```

## Plotting

+++

Similar to Pandas, there is a `plot` method, which can be used for different plot types:

```{code-cell} ipython3
xr_array = xr.open_rasterio("./data/gent/raster/2020-09-17_Sentinel_2_L1C_B0408.tiff")
xr_array = xr_array.assign_coords(band=("band", ["b4", "b8"]))
```

It supports both 2 dimensional (e.g. line) as 3 (e.g. imshow, pcolormesh) dimensional plots. When just using `plot`, xarray will do a _best guess_ on how to plot the data. However being explicit `plot.line`, `plot.imshow`, `plot.pcolormesh`, `plot.scatter`,...  gives you more control.

```{code-cell} ipython3
xr_array.sel(band="b4").plot();  # add .line() -> ValueError: For 2D inputs, please specify either hue, x or y.
```

```{code-cell} ipython3
xr_array.sel(x=420000, method="nearest").plot.line(hue="band");
```

`facetting` splits the data in subplots according to a dimension, e.g. `band`

```{code-cell} ipython3
xr_array.sel(x=420000, method="nearest").plot.line(col="band");  # row="band"
```

Use the `robust` option when there is a lack of visual difference. This will use the 2nd and 98th percentiles of the data to compute the color limits. The arrows on the color bar indicate that the colors include data points outside the bounds.

```{code-cell} ipython3
ax = xr_array.sel(band="b4").plot(cmap="Reds", robust=True, figsize=(12, 5))
ax.axes.set_aspect('equal')
```

Compare data variables within a `xarray Dataset`:

```{code-cell} ipython3
xr_data = xr_array.to_dataset(dim="band")
xr_data.plot.scatter(x="b4", y="b8", s=2)
```

Calculating and plotting the NDVI in three classes illustrates the options of the `imshow` method:

```{code-cell} ipython3
xr_data["ndvi"] = (xr_data["b8"] - xr_data["b4"])/(xr_data["b8"] + xr_data["b4"])
xr_data["ndvi"].plot.imshow(levels=[-1, 0, 0.3, 1.], colors=["gray", "yellowgreen", "g"])
```

# Let's practice!

+++

The data set for the following exercises is from [Argo floats](https://argo.ucsd.edu/), an international collaboration that collects high-quality temperature and salinity profiles from the upper 2000m of the ice-free global ocean and currents from intermediate depths.

These data do not represent full coverage image data (like remote sensing images), but measurements of salinity and temperature as a function of water `level` (related to the pressure). Each measurements happens at a given `date` on a given location (`lon`/`lat`).

```{code-cell} ipython3
import xarray as xr
argo = xr.load_dataset("./data/argo_float.nc")
```

```{code-cell} ipython3
argo
```

The bold font (or * symbol in plain text output version) in the coordinate representation above indicates that level and date are 'dimension coordinates' (they describe the coordinates associated with data variable axes) while lon and lat are 'non-dimension coordinates'. We can make any variable a non-dimension coordinate.

+++

Let's plot the coordinates of the available measurements and add a background map using [contextly](https://contextily.readthedocs.io/en/latest/index.html):

```{code-cell} ipython3
import contextily as cx
from pyproj import CRS
crs = CRS.from_epsg(4326)

fig, ax = plt.subplots(figsize=(6, 6))
argo.plot.scatter(x='lon', y='lat', ax=ax, color='red', s=2)

# Custom adjustments of the limits, as we are in the middle of the ocean
xmin, xmax = ax.get_xlim()
ymin, ymax = ax.get_ylim()
ax.set_xlim(xmin*1.5, xmax*0.5)
ax.set_ylim(ymin*0.5, ymax*1.5)

cx.add_basemap(ax, crs=crs.to_string())
```

<div class="alert alert-success">

**EXERCISE**:

Add a new variable to the `argo` data set, called `temperature_kelvin`, by converting the temperature to Kelvin. 
    
Degrees Kelvin = degrees celsius + 273.
   
<details>
    
<summary>Hints</summary>

* Remember that xarray works as Numpy and relies on the same broadcasting rules.

</details>    
    
</div>

```{code-cell} ipython3
:clear_cell: true

argo["temperature_kelvin"] = argo["temperature"] + 273.15
```

<div class="alert alert-success">

**EXERCISE**:

The water level classes define different water depths. The pressure is a proxy for the water depth. Verify the relationship between the pressure and the level using a scatter plot. Does a larger value for the level represent deeper water depths or not?
    
<details><summary>Hints</summary>

* If you get the error `ValueError: Dataset.plot cannot be called directly. Use an explicit plot method, e.g. ds.plot.scatter(...)`, read the message and do what it says.

</details>    
    
</div>

```{code-cell} ipython3
:clear_cell: true

argo.plot.scatter(x="pressure", y="level")
```

<div class="alert alert-success">

**EXERCISE**:

Assume that buoyancy is defined by the following formula:
    
$$g \cdot ( 2\times 10^{-4} \cdot T - 7\times 10^{-4} \cdot P )$$

With:
- $g$ = 9.8
- $T$ = temperature
- $P$ = pressure

Calculate the buoyancy and add it as a new variable `buoyancy` to the `argo` data set. 

Make a 2D (image) plot with the x-axis the date, the y-axis the water level and the color intensity the buoyancy. As the level represents the depth of the water, it makes more sense to have 0 (surface) at the top of the y-axis: switch the y-axis direction.
    
<details><summary>Hints</summary>

* Remember that xarray works as Numpy and relies on the same broadcasting rules.
* The `imshow` method does not work on irregular intervals. Matplotlib and xarray also have `pcolormesh`.    
* Look for options [in the xarray documentation](http://xarray.pydata.org/en/stable/plotting.html#other-axes-kwargs) to control the axis direction. (The `ax.invert_yaxis()` Matplotlib function is not supported for pcolormesh)
    
</details>    
    
</div>

```{code-cell} ipython3
:clear_cell: true

argo["buoyancy"] = 9.8 * (2e-4 * argo["temperature"] - 7e-4 * argo["pressure"])
```

```{code-cell} ipython3
:clear_cell: true

argo["buoyancy"].plot(yincrease=False)  # xarray decides the plot type when no specific method is used
```

```{code-cell} ipython3
:clear_cell: true

# More explicit version defining the x and y axis
argo["buoyancy"].plot.pcolormesh(x="date", y="level", yincrease=False)  # pcolormesh instead of imshow
```

<div class="alert alert-success">

**EXERCISE**:

Make a line plot of the salinity as a function of time at level 10
   
<details><summary>Hints</summary>

Break it down into different steps and chain the individual steps:
    
* From the argo data set, select the variable `salinity`. This is similar to selecting a column in Pandas.
* Next, use the `sel` method to select the `level=10`
* Next, use the `plot.line()` method.

</details>    
    
</div>

```{code-cell} ipython3
:clear_cell: true

argo["salinity"].sel(level=10).plot.line()
```

<div class="alert alert-success">

**EXERCISE**:

- Make a line plot of the temperature as a function of time for the levels 10, 20 and 30 at the same graph 
- Make a second line plot with each of the 3 levels (10, 20, 30) in a different subplot. 
   
<details><summary>Hints</summary>

Break it down into different steps and chain these individual steps:
    
* From the argo data set, select the variable `temperature`. This is similar to selecting a column in Pandas.
* Next, use the `sel` method to select the levels 10, 20 and 30.
* Next, use the `plot.line()` method, but make sure the `hue` changes for each level
    
For the subplots, check the [facetting documentation](http://xarray.pydata.org/en/stable/plotting.html#faceting) of xarray. 

</details>    
    
</div>

```{code-cell} ipython3
:clear_cell: true

argo["temperature"].sel(level=[10, 20, 30]).plot.line(hue="level");
```

```{code-cell} ipython3
:clear_cell: true

argo["temperature"].sel(level=[10, 20, 30]).plot.line(col="level");
```

<div class="alert alert-success">

**EXERCISE**:

You wonder how the temperature evolves with increasing latitude and what the effect is of the depth (level):

- Create a scatter plot of the `level` as a function of the `temperature` colored by the `latitude`. 
    
- As a further exploration step, pick a subset of levels 1, 10, 25, and 50 and create a second scatter plot with in the x-axis the latitude of the measurement and in the y-axis the temperature. To compare the effect of the different levels, give each level a separate subplot next to each other.
   
<details><summary>Hints</summary>

* In a scatter plot, the color or hue can be linked to a variable.
* From the argo data set, use the `sel` method to select the levels 1, 10, 25, and 50.
* For the second scatter plot, but make sure the `col` changes for each `level` and define which variables need to go to which axis.

</details>    
    
</div>

```{code-cell} ipython3
:clear_cell: true

argo.plot.scatter(x="temperature", y="level", hue="lat", s=2)
```

```{code-cell} ipython3
:clear_cell: true

argo.sel(level=[1, 5, 25, 50]).plot.scatter(x="lat", y="temperature", col="level")
```

<div class="alert alert-success">

**EXERCISE**:

Make an image plot of the temperature as a function of time. Divide the colormap in 3 discrete categories:
    
* x < 5
* 5 < x < 15
* x > 15
    
Choose a custom colormap and adjust the label of the colorbar to `'Temperature (°C)'`
   
<details>
    
<summary>Hints</summary>

- Check the help of the `plot` function or the [xarray documentation](http://xarray.pydata.org/en/stable/plotting.html#discrete-colormaps) on discrete colormaps.
- Adjustments to the colorbar settings can be defined with the `cbar_kwargs` as a dict. Adjust the `label` of the colorbar.    

</details>    
    
</div>

```{code-cell} ipython3
:clear_cell: true

argo.temperature.plot.pcolormesh(yincrease=False, 
                                 cbar_kwargs={'label': 'Temperature (°C)'},
                                 cmap='Reds',
                                 levels=[5, 15]
                                );
```

<div class="alert alert-success">

**EXERCISE**:

Calculate the average salinity and temperature as a function of level over the measurements taken between 2012-10-01 and 2012-12-01. 

Make a separate line plot for each of them. Define the figure and 2 subplots first with Matplotlib. 
    
<details><summary>Hints</summary>

* xarray supports to query dates using a string representation.
* Use the `slice` operator within the `sel` to select a range of the data.
* Whereas in Numpy we used `axis` in reduction functions, xarray uses the `dim` name.
* Also for line plots you can define which dimension should be on the x-axis and which on the y-axis by providing the name.  
* Use `fig, (ax0, ax1) = plt.subplots(1, 2)` to create subplots.
</details>    
    
</div>

```{code-cell} ipython3
:clear_cell: true

subset_mean = argo.sel(date=slice('2012-10-01', '2012-12-01')).mean(dim="date")
```

```{code-cell} ipython3
:clear_cell: true

fig, (ax0, ax1) = plt.subplots(1, 2)
subset_mean["salinity"].plot.line(y="level", yincrease=False, ax=ax0)
subset_mean["temperature"].plot.line(y="level", yincrease=False, ax=ax1)
plt.tight_layout()
```

## Pandas for multiple dimensions...

```{code-cell} ipython3
argo = xr.load_dataset("./data/argo_float.nc")
```

If we are interested in the _average over time_ for each of the levels, we can use a reducton function to get the averages of each of the variables at the same time:

```{code-cell} ipython3
argo.mean(dim=["date"])
```

But if we wanted the _average for each month of the year_ per level, we would first have to __split__ the data set in a group for each month of the year, __apply__ the average function on each of the months and __combine__ the data again. 

We already learned about the [split-apply-combine](https://pandas.pydata.org/pandas-docs/stable/user_guide/groupby.html) approach when using Pandas. The syntax of Xarray’s groupby is almost identical to Pandas! 

+++

First, extract the month of the year (1-> 12) from each of the date coordinates:

```{code-cell} ipython3
argo.date.dt.month  # The coordinates is a Pandas datetime index
```

We can use these arrays in a groupby operation:

```{code-cell} ipython3
argo.groupby(argo.date.dt.month)
```

Xarray also offers a more concise syntax when the variable you're grouping on is already present in the dataset. This is identical to the previous line:

```{code-cell} ipython3
argo.groupby("date.month")
```

Next, we apply an aggregation function _for each of the months_ over the `date` dimension in order to end up with: _for each month of the year, the average (over time) for each of the levels_:

```{code-cell} ipython3
argo.groupby("date.month").mean(dim="date")        #["temperature"].sel(level=1).to_series().plot.barh()
```

Another (alike) operation - specifically for time series data - is to `resample` the data to another time-aggregation. For example, resample to monthly (`1M`) or yearly (`1Y`) median values:

```{code-cell} ipython3
argo.resample(date="1M").median()  # 1Y
```

```{code-cell} ipython3
argo["salinity"].sel(level=1).plot.line(x="date");
argo["salinity"].resample(date="1M").median().sel(level=1).plot.line(x="date");  # 1Y
```

A similar, but different functionality is `rolling` to calculate rolling window aggregates:

```{code-cell} ipython3
argo.rolling(level=10, center=True).std()
```

```{code-cell} ipython3
argo["salinity"].sel(date='2012-10-31').plot.line(y="level", yincrease=False, color="grey");
argo["salinity"].sel(date='2012-10-31').rolling(level=10, center=True).median().plot.line(y="level", yincrease=False, linewidth=3, color="crimson");
plt.legend(), plt.title("");
```

<div class="alert alert-info" style="font-size:120%">

**REMEMBER**: <br>

The [xarray `groupby`](http://xarray.pydata.org/en/stable/groupby.html) with the same syntax as Pandas implements the __split-apply-combine__ strategy. Also [`resample`](http://xarray.pydata.org/en/stable/time-series.html#resampling-and-grouped-operations) and [`rolling`](http://xarray.pydata.org/en/stable/computation.html?highlight=rolling#rolling-window-operations) are available in xarray.
    
__Note:__ Xarray adds a [`groupby_bins`](http://xarray.pydata.org/en/stable/generated/xarray.Dataset.groupby_bins.html#xarray.Dataset.groupby_bins) convenience function for binned groups (instead of each value).

</div>

+++

---------------

__Note:__ Values are only read from disk when needed. For example, the following statement only reads the coordinate information and the metadata. The data itself is not yet loaded:

```{code-cell} ipython3
data_file = "./data/gent/raster/2020-09-17_Sentinel_2_L1C_True_color.tiff"
```

```{code-cell} ipython3
gent = xr.open_rasterio(data_file)
gent
```

`load()` will explicitly load the data into memory:

```{code-cell} ipython3
xr.open_rasterio(data_file).load()
```

Acknowledgements and great thanks to https://earth-env-data-science.github.io for the inspiration, data and examples.
