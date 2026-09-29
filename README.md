# Maps Extension Specification

- **Title:** Maps
- **Identifier:** <https://stac-extensions.github.io/maps/v1.0.0/schema.json>
- **Field Name Prefix:** none
- **Scope:** Catalog, Collection, Item
- **Extension [Maturity Classification](https://github.com/radiantearth/stac-spec/tree/master/extensions/README.md#extension-maturity):** Proposal
- **Owner**: @m-mohr @emmanuelmathot

This document explains the Maps Extension to the [SpatioTemporal Asset Catalog](https://github.com/radiantearth/stac-spec) (STAC) specification.

A map combines several layers into a single view with a projection and a default extent.
The layers can come from assets, web map links or renders,
and a stylesheet or a render can be applied to the data of an asset or a service.
Multiple maps can be provided, e.g. to offer different visualizations of the same data.

This extension doesn't depend on other extensions.
It uses the web map links of the [Web Map Links Extension](https://github.com/stac-extensions/web-map-links)
and the renders of the [Render Extension](https://github.com/stac-extensions/render) if a document provides them.

- Examples:
  - [Item example](examples/item.json): Shows maps that combine assets, web map links, renders and stylesheets in a STAC Item
  - [Collection example](examples/collection.json): Shows a map with a basemap and a web map link in a STAC Collection
- [JSON Schema](json-schema/schema.json)
- [Changelog](./CHANGELOG.md)

## Fields

The fields in the table below can be used in these parts of STAC documents:

- [x] Catalogs
- [x] Collections
- [x] Item Properties
- [ ] Assets (for both Collections and Items, incl. Item Asset Definitions in Collections and Asset Templates)
- [x] Links (incl. Link Templates), only [`id`](#link-identifiers)
- [ ] Bands

| Field Name | Type                                     | Description |
| ---------- | ---------------------------------------- | ----------- |
| maps       | Map\<string, [Map Object](#map-object)> | **REQUIRED**. Named maps. The keys SHOULD match `^[A-Za-z0-9_-]+$`. |

In Catalogs and Collections, `maps` is a top-level field. In Items, it's part of the `properties`.

### Map Object

| Field Name    | Type                                | Description |
| ------------- | ----------------------------------- | ----------- |
| title         | string                              | A title for the map, e.g. for a map picker. |
| description   | string                              | A description of the map. [CommonMark 0.29](https://commonmark.org/) syntax MAY be used for rich text representation. |
| roles         | \[string]                           | `default` marks the map that clients show by default. |
| proj:code     | string\|null                        | The projection of the map, as defined in the [Projection Extension](https://github.com/stac-extensions/projection). |
| proj:wkt2     | string\|null                        | The projection of the map, as defined in the [Projection Extension](https://github.com/stac-extensions/projection). |
| proj:projjson | object\|null                        | The projection of the map, as defined in the [Projection Extension](https://github.com/stac-extensions/projection). |
| bbox          | \[number]                           | The default extent of the map in WGS 84 (EPSG:4326), with 4 or 6 numbers as for the `bbox` of Items. |
| dimensions    | Map\<string, \*>                    | Default values for the non-spatial dimensions of all layers (e.g. a common `time`). |
| layers        | \[[Layer Object](#layer-object)]    | **REQUIRED**. The layers of the map, from bottom to top. Can't be empty. |

If no projection is provided, clients choose one, e.g. Web Mercator.
If multiple projection fields are provided, they MUST describe the same projection.

If no `bbox` is provided, clients use the spatial extent of the STAC entity.

**dimensions**: The keys are the names of the dimensions, as used in the [Datacube Extension](https://github.com/stac-extensions/datacube)
(`cube:dimensions`) or by the services (e.g. `TIME` in WMS). The values apply wherever a layer's source doesn't set its own value.

### Layer Object

A layer has exactly one **source** and optionally a **style**.

| Field Name     | Type                                                          | Description |
| -------------- | ------------------------------------------------------------- | ----------- |
| asset          | string                                                        | Source: The key of an asset. |
| link           | string\|Link Object\|Link Template Object                     | Source: The [`id`](#link-identifiers) of a link or link template in the document, or an embedded link for layers from outside of the document (e.g. a basemap). |
| render         | string                                                        | Source: The key of a render. The render provides its own data and portrayal. |
| style          | [Style Reference Object](#style-reference-object)             | The portrayal that is applied to the data of an `asset` or `link` source. |
| title          | string                                                        | A title for the layer, e.g. for a layer switcher. Overrides the title of the source. |
| opacity        | number                                                        | The opacity of the layer, between 0 (transparent) and 1 (opaque). Defaults to `1`. |
| visible        | boolean                                                       | Whether the layer is visible when the map loads. Defaults to `true`. |
| min_resolution | number                                                        | The finest resolution at which the layer is shown, see [resolution ranges](#resolution-ranges). |
| max_resolution | number                                                        | The coarsest resolution at which the layer is shown, see [resolution ranges](#resolution-ranges). |

Each layer MUST provide exactly one of `asset`, `link` and `render`.
The keys and `id`s are looked up in the document first and then in the parent Collection.
For example, an Item can use a render that its Collection defines for the corresponding `item_assets`.

**link**: Web map links from the document are referenced by their `id`, which avoids duplicating them.
Embedding a link is meant for layers that are not part of the document, e.g. a basemap from a third party.
Embedded links follow the definitions of the web map link in use,
e.g. from the [Web Map Links Extension](https://github.com/stac-extensions/web-map-links).

**style**:

- It MUST NOT be provided for a `render` source, as a render has its own portrayal.
- For a `link` source, it can only be provided for services that deliver data instead of images,
  e.g. vector tiles through XYZ, TileJSON or PMTiles.

**min_resolution** and **max_resolution**: If the source of the layer is a link that indicates a resolution range itself,
clients only show the layer where both ranges allow it.

### Style Reference Object

| Field Name | Type                | Description |
| ---------- | ------------------- | ----------- |
| link       | string\|Link Object | The `id` of a stylesheet link in the document, or an embedded stylesheet link. |
| render     | string              | The key of a render that is applied to the data of the layer's source. |

Each Style Reference Object MUST provide exactly one of `link` and `render`.

**link**: Stylesheet links use the relation type `stylesheet`, which is
[registered by IANA](https://www.iana.org/assignments/link-relations/link-relations.xhtml).
The `type` is REQUIRED and identifies the encoding of the stylesheet, preferably with the media types of
[OGC API - Styles](https://docs.ogc.org/DRAFTS/20-009.html), e.g. `application/vnd.mapbox.style+json` or `application/vnd.ogc.sld+xml`.

**render**: Only renders that don't reference any data themselves can be applied to another source.

### Possible combinations

The following table shows the possible combinations of sources and styles:

| Layer                                                             | Shows |
| ----------------------------------------------------------------- | ----- |
| `{"asset": "visual"}`                                             | The asset with the default portrayal of the client, e.g. an RGB image or a default vector style. |
| `{"asset": "fields", "style": {"link": "fields-outline"}}`        | The asset, styled by a stylesheet. |
| `{"asset": "fields", "style": {"render": "fields"}}`              | The asset, styled by a render. |
| `{"render": "ndvi"}`                                              | A render with its own data, rendered by the client or delivered by a web map link that references the render. |
| `{"link": "topo-wms"}`                                            | A service as it's delivered. |
| `{"link": "fields-pmtiles", "style": {"link": "fields-outline"}}` | The data of a service, styled by a stylesheet. |
| `{"link": "fields-pmtiles", "style": {"render": "fields"}}`       | The data of a service, styled by a render. |
| `{"link": {"rel": "xyz", "href": "…"}}`                           | A layer from outside of the document. |

### Resolving layers

Clients resolve each layer as follows:

1. `asset`: Read the asset. Apply the `style` if provided, otherwise use the default portrayal of the client.
2. `link`: Load the service. Apply the `style` to its data if provided.
3. `render`: If the client supports the render and can read its data, render it on the client.
   Otherwise, use a web map link that indicates that it delivers the render (e.g. through the field `render`)
   and that the client supports. If multiple links match, they are equivalent and the client can choose any of them.
4. If none of the above works, skip the layer and inform the user.

Clients that don't support this extension usually fall back to showing the assets, renders or web map links directly.

### Resolution ranges

`min_resolution` and `max_resolution` define the range of resolutions in which a layer is shown.
Clients SHOULD hide the layer outside of this range.
If both are provided, `min_resolution` must be lower than or equal to `max_resolution`.

The resolution is the **distance on the ground in meters per display (CSS) pixel**, measured at the center of the
spatial extent of the STAC entity (`bbox` in Items, the first bounding box of the spatial extent in Collections).
This makes the value independent of the projection of the map.
Clients convert the value into the units of the map projection, e.g. with `getPointResolution` in OpenLayers.

For example, a `max_resolution` of `1000` for data centered at 48° North converts to:

| Map projection            | Resolution in map units |
| ------------------------- | ----------------------- |
| EPSG:25832 (UTM zone 32N) | ≈ 1000 m per pixel |
| EPSG:3857 (Web Mercator)  | ≈ 1000 / cos(48°) ≈ 1494 m per pixel |
| EPSG:4326 (WGS 84)        | ≈ 0.0134° per pixel horizontally, ≈ 0.0090° per pixel vertically |

The ground resolution varies over large extents, so the range is approximate far from the center.
Clients MAY evaluate it at the center of the current view instead,
which is also the fallback for Catalogs and other cases where no spatial extent is available.

The data-side counterpart is `raster:spatial_resolution` of the [Raster Extension](https://github.com/stac-extensions/raster),
which is also measured in meters on the ground.
For example, a layer with categorical data may set its `min_resolution` to the `raster:spatial_resolution`
so that it's not shown beyond the resolution of the data.
`gsd` is not suitable to derive the range as it describes the sensor, not the pixels of the data.

## Link identifiers

This extension adds the following field to the
[Link Object](https://github.com/radiantearth/stac-spec/tree/master/commons/links.md#link-object)
and the Link Template Object of the [Link Templates Extension](https://github.com/stac-extensions/link-templates):

| Field Name | Type   | Description |
| ---------- | ------ | ----------- |
| id         | string | An identifier for the link, which MUST be unique across all `links` and `linkTemplates` of the document. **REQUIRED** for links that are referenced by a map. |

Links are never referenced by their position in the array,
as the order can change, e.g. when a STAC API adds or regenerates links such as `self`, `root` or `parent`.

## Relation to other specifications

- [Render Extension](https://github.com/stac-extensions/render):
  A render describes how data is portrayed. A map decides which renders are shown together, and where.
  Visibility ranges are therefore defined for the map layers, not for renders.
- [Web Map Links Extension](https://github.com/stac-extensions/web-map-links):
  Web map links describe services. A map references them as layers by their `id`.
  A web map link can indicate that it delivers a render (e.g. through the field `render`),
  so that clients can choose between rendering on the client and using the service.
- [Link Templates Extension](https://github.com/stac-extensions/link-templates):
  Link templates are referenced by their `id` like links.
- [Projection Extension](https://github.com/stac-extensions/projection):
  The fields `proj:code`, `proj:wkt2` and `proj:projjson` are used to define the projection of a map.
- [OGC API - Maps](https://docs.ogc.org/DRAFTS/20-058.html):
  The map resources of OGC API - Maps (e.g. `/map` and `/styles/{styleId}/map`) are services that deliver images.
  They are described as web map links and can be used as layers in the maps of this extension.
- [OGC API - Styles](https://docs.ogc.org/DRAFTS/20-009.html):
  Stylesheet links follow OGC API - Styles, i.e. the relation type `stylesheet` and the media types for the encodings.

## Contributing

All contributions are subject to the
[STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).
For contributions, please follow the
[STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md) Instructions
for running tests are copied here for convenience.

### Running tests

The same checks that run as checks on PRs are part of the repository and can be run locally to verify that changes are valid.
To run tests locally, you'll need `npm`, which is a standard part of any [node.js installation](https://nodejs.org/en/download/).

First you'll need to install everything with npm once. Just navigate to the root of this repository and on
your command line run:

```bash
npm install
```

Then to check markdown formatting and test the examples against the JSON schema, you can run:

```bash
npm test
```

This will spit out the same texts that you see online, and you can then go and fix your markdown or examples.

If the tests reveal formatting problems with the examples, you can fix them with:

```bash
npm run format-examples
```
