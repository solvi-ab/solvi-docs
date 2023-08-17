---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell

toc_footers:


includes:


search: false
---

# Introduction

Solvi API is organized around REST and allows for programmatic access to resources like users and projects. Below you will find detailed information on how to access the API and which operations are currently supported.

In Solvi, a Project represents a single upload of multiple images. Projects can be grouped under Fields, which in turn are grouped into Farms to make it easier to organize and share data. Current API implementation allows to create and fetch projects for the specific user, as well as upload and stitch images into maps. Once processed, the outputs such as orthomosaics and elevation maps can either be accessed through the API or the user can be redirected to Solvi to view and analyze the processed imagery.

# Authentication

> Example API key request:

```shell
curl "api_endpoint_here"
  -H "X-Api-Key: <your-api-key>"
```
> Make sure to replace `<your-api-key>` with your API key.

To use Solvi API you first need an *API key*. You will get an API key from Solvi, [contact us](mailto:support@solvi.nu) to get your key. If you for some
reason need to revoke a key, you can also contact us.

Please note that the API key should be kept secret and not be exposed to end-users: for example, never send the API key to a web browser.

With the API key, you can access general API methods, for example, to [register new users](#register-new-user) in Solvi, and create user-specific tokens.

Requests requiring an API key should add the `X-Api-Key` header:

`X-Api-Key: <your-api-key>`

> Example user-specific request:

```shell
curl "api_endpoint_here"
  -H "Authorization: Bearer <user-specific-token>"
```
> Make sure to replace `<user-specific-token>` with your token.

Instead of using the API key, requests that act on behalf of a specific user, like creating or getting projects, should use a user-specific token to authenticate as the user. When using a user-specific token, *do not* include the API key in the same request. A user-specific token has a limited lifetime and is less security sensitive compared to the API key. For example, a user token can be sent to and used directly from for example a browser, or for passwordless authentication when the user is redirected from your portal to Solvi. A user-specific token can be used for a limited amount of time (currently 24 hours) before it expires. A user-specific token is created by using the [endpoint to create a user-specific token](#generate-user-specific-token).

User specific requests should use the `Authorization` header *instead of* the `X-Api-Key` header:

`Authorization: Bearer <user-specific-token>`

# Users

## Register new user

> Example request:

```shell
curl -X POST
  -H "X-Api-Key: <your-api-key>"
  -H "Content-Type: application/json"
  -d '{ "user": { "email": "john@example.com", "password": "password", "first_name": "John", "last_name": "Doe"}}'
  "https://solvi.ag/api/v1/users"
```

> Example response:

```json
  {
    "status": "success",
    "user_id": 182
  }
```

This endpoint registers a new user. If successful, the response will return `user_id` that you would use later to generate user-specific tokens.

### HTTP Request

`POST https://solvi.ag/api/v1/users`

### Parameters

Parameter |  | Description
--------- | ------- | -----------
email | required | Email
password | required | Password
first_name | required | First name
last_name | required | Last name

<aside>
Parameters above must be wrapped into `user` attribute and sent as JSON payload in POST request, see the example.
</aside>

## Generate user-specific token

> Example request:

```shell
curl -X GET
  -H "X-Api-Key: <your-api-key>"
  "https://solvi.ag/api/v1/users/<user_id>/token"
```

> Example response:

```json
  {
    "status": "success",
    "user_id": 26,
    "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoyNiwiZXhwIjo0OCwiZXhwIjoxNTAxMjU2NzAyfQ.cu5zIye7ubBhv7YsFIxXkO0E_W0hG0VrlOTQx6L3b3c"
  }
```

This endpoint gives a token for the specific user that should be used to create and retrieve user projects. Every request will generate na ew token. The token is valid for 24 hours.

### HTTP Request

`GET https://solvi.ag/api/v1/users/<user_id>/token`

### Parameters

Parameter |  | Description
--------- | ------- | -----------
user_id | required | User ID given when user is created

# Attaching custom metadata

When using the API, it is common that the farms, fields and projects in Solvi also exist in your own
systems, and you need information to track which entity in your system a Solvi entity maps to. There
are many ways to accomplish this, but one way is to attach metadata to the Solvi entities. Farms, fields
and projects created through the API all support attaching arbitrary data to them. Solvi does not use this data, so you can put anything here, as long as it can be encoded in JSON.

Endpoints to create or update farms, fields and projects all accept an optional parameter `metadata` for this purpose.

Endpoints that return farms, fields and projects also return the metadata under the `metadata` property if set, or `null` otherwise.

# Farms

## Create farm

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{"farm": {"name": "My second farm"}}'
  "https://solvi.ag/api/v1/farms"
```

> Example response:

```json
  {
    "status": "success",
    "farm_id": 793
  }
```

Creates a new farm.

### HTTP Request

`POST https://solvi.ag/api/v1/farms`

### Parameters

Parameter |             | Description
--------- | ----------- | -----------
name      | required    | Name of the the farm
metadata  | optional    | Optional arbitrary metadata

## Get farms

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://solvi.ag/api/v1/farms"
```

> Example response:

```json
    [
      {
        "name": "My farm",
        "created_at": "2019-02-08T09:18:37.655Z",
        "fields": [
          {
            "id": 2657,
            "name": "Veddige",
            "created_at": "2019-03-08T04:05:38.628Z",
            "metadata": null
          }
        ],
        "metadata": null
      }
    ]
```

Gets all user farms and a list of fields related to each farm.

### HTTP Request

`GET https://solvi.ag/api/v1/farms`


# Fields

## Create field

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{"field": { "name": "Winter Wheat", "geom": "{\"type\":\"Polygon\",\"coordinates\":[[[100.0, 0.0],[101.0, 0.0],[101.0, 1.0],[100.0, 1.0],[100.0, 0.0]]]}"}}'
  "https://solvi.ag/api/v1/fields"
```

> Example response:

```json
  {
    "status": "success",
    "field_id": 793
  }
```

Creates a new field. Field boundaries can be provided as Polygon or MultiPolygon in GeoJSON format. The farm to organize the field under can also be provided. The response contains `field_id` which can be later used to relate projects to the specific field.

### HTTP Request

`POST https://solvi.ag/api/v1/fields`

### Parameters

Parameter |             | Description
--------- | ----------- | -----------
name      | required    | Name of the the field
geom      | optional    | Boundaries of the field as a Polygon or Multipolygon in [GeoJSON format](https://geojson.org/geojson-spec.html#introduction) and EPSG:4326 coordinate system(lonlat)
farm_id   | optional    | The id of the farm to put the field under; if not specified, the user's last created farm is used
metadata  | optional    | Optional arbitrary metadata



## Get fields

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://solvi.ag/api/v1/fields"
```

> Example response:

```json
    [
      {
          "id": 199,
          "name": "wheat field",
          "created_at": "2018-02-27T12:45:05.550Z",
          "projects": [
              {
                  "project_id": 1291,
                  "status": "processed",
                  "name": "27-FEB-2018",
                  "field_name": "Wheat field",
                  "survey_date": "2018-02-26T14:19:36.000Z",
                  "upload_date": "2018-02-27T12:45:05.556Z",
                  "url": "https://solvi.ag/projects/1291",
                  "thumbnail_url": "https://solvi.ag/projects/1291/thumbnail.png",
                  "metadata": null
              }
          ],
          "metadata": null
      }
    ]
```

Gets all user fields and a list of projects related to each field.

### HTTP Request

`GET https://solvi.ag/api/v1/fields`


# Projects

## Create project

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{ "type": "overlapping", "field_id": "field_1", "field_name": "Wheat Field", "field_geom": "{\"type\":\"Polygon\",\"coordinates\":[[[100.0, 0.0],[101.0, 0.0],[101.0, 1.0],[100.0, 1.0],[100.0, 0.0]]]}" }'
  "https://solvi.ag/api/v1/projects"
```

> Example response:

```json
  {
    "status": "success",
    "project_id": 142,
    "project_url": "https://solvi.ag/projects/142/photos/upload",
    "imagery_upload_data": {
      "url":"https://solvi-projects-dev.s3.eu-west-1.amazonaws.com",
      "fields": {
        [...]
      },
      "key_prefix":"uploads/26c1ce11-c9bf-4825-821c-72e9f600a6cf/originals/"
    },
  }
```

This endpoint creates a new project which is required before imagery upload. In response, you will receive URL to upload-page for the newly created project where the user can be redirected.

There are multiple *types* of projects: `overlapping` (the default), `stitched` and `scouting`:

* `overlapping` should be used if you have a number of photos taken which should be stitched into a map by Solvi; this is the default and what you normally use
* `stitched` can be used if you already have a single [GeoTIFF](https://en.wikipedia.org/wiki/GeoTIFF) that have been stitched by another system
* In `scouting`, images will not be stitched and instead shown as separate, individual images without further processing 

Projects can be connected to a Field. When multiple projects are related to the same Field, they appear in the same map view when data is processed. This allows for easier navigation between imagery over the same Field and over the time data comparison.

Fields can be either be:

* [created beforehand](#create-field); for this case, set the `field_id` parameter to the previously created field's `id`
* created on the fly by sending `field_name` and `field_geom` (optional), which will create a new field for the project; do not include `field_id`

Optionally, a project can be created with a so-called *webhook* that will be called every time the status of the project changes. This makes it possible for integration to for example react when a project finishes processing, without having to use polling to check the project's status. To add a webhook, specify the URL to be called with the `status_webhook` parameter. See the section on [webhooks](#webhooks) for details.

The project's crop, survey date and altitude, data which will be displayed in Solvi's UI, can also be submitted.

### HTTP Request

`POST https://solvi.ag/api/v1/projects`

### Parameters

Parameter  |             | Description
---------- | ----------- | -----------
type       | optional    | The type of imagery for this project: `overlapping`, `stitched` or `scouting`; default is `overlapping`
field_id   |             | Solvi field id as returned from [Fields API](#fields)
field_name |             | Name of the the field
field_geom | optional    | Boundaries of the field as a polygon in [GeoJSON format](https://geojson.org/geojson-spec.html#introduction) and EPSG:4326 coordinate system(lonlat)
status_webhook | optional | A URL to be called when the project's status changes
webhook_secret | optional | A token to use to sign webhook requests, see [webhooks](#webhooks) for details
survey_date | optional   | Date survey was conducted, formatted as YYYY-MM-DD
crop       | optional    | The documented crop; free form string
altitude   | optional    | Altitude the imagery was taken from in meters
metadata   | optional    | Optional arbitrary metadata

## Update project data

> Example request:

```shell
curl -X PUT
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{
    "survey_date": "2022-08-18"
  }'
```

> Example response:

```json
{
    "status": "ok"
}
```

### HTTP Request

`PUT https://solvi.ag/api/v1/projects/<project_id>`

### Parameters

Parameter  |             | Description
---------- | ----------- | -----------
survey_date | optional   | Date survey was conducted, formatted as YYYY-MM-DD
crop       | optional    | The documented crop; free form string
altitude   | optional    | Altitude the imagery was taken from in meters
metadata   | optional    | Optional arbitrary metadata

This endpoint allows updating survey date, crop, altitude and metadata for a project.

## Upload local imagery

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://solvi.ag/api/v1/projects/<project_id>/begin_upload"
```

> Example response:

```json
{
  "status": "ok",
  "imagery_upload_data": {
    "url": "https://solvi-projects.s3.eu-west-1.amazonaws.com",
    "fields": {
      "x-amz-storage-class": "STANDARD_IA",
      "policy": "eyJleHBpcmF0aW9uIjoiMjAyMC0wOC0wN1QwNzozMjo1MVoiLCJjb25kaXRpb25zIjpbeyJidWNrZXQiOiJzb2x2aS1wcm9qZWN0cy1kZXYifSxbInN0YXJ0cy13aXRoIiwiJGtleSIsInVwbG9hZHMvMjZjMWNlMTEtYzliZi00ODI1LTgyMWMtNzJlOWY2MDBhNmNmL29yaWdpbmFscy8iXSxbInN0YXJ0cy13aXRoIiwiJENvbnRlbnQtVHlwZSIsIiJdLHsieC1hbXotc3RvcmFnZS1jbGFzcyI6IlNUQU5EQVJEX0lBIn0seyJ4LWFtei1jcmVkZW50aWFsIjoiQUtJQVVWTUhLR1RRRldMTlkyTUgvMjAyMDA4MDUvZXUtd2VzdC0xL3MzL2F3czRfcmVxdWVzdCJ9LHsieC1hbXotYWxnb3JpdGhtIjoiQVdTNC1ITUFDLVNIQTI1NiJ9LHsieC1hbXotZGF0ZSI6IjIwMjAwODA1VDA3MzI1MVoifV18",
      "x-amz-credential": "AKIAUVMHKGTQFWLNY2MX/20200805/eu-west-1/s3/aws4_request",
      "x-amz-algorithm": "AWS4-HMAC-SHA256",
      "x-amz-date": "20200805T073251Z",
      "x-amz-signature": "5494b8f7ca5c78c45daf7d8099f3c4b93e141567d3691dbec3dcc0f6fd2e0181"
    },
    "key_prefix": "uploads/26c1ce11-c9bf-4825-821c-72e9f600a6cf/originals/"
  }
}
```

After a project has been created, imagery can be uploaded to it by sending the `begin_upload` request. The response includes information required to upload
imagery for the project.

> Example JavaScript function to POST an image using `imagery_upload_data`
> retrieved from the example above:

```js
function uploadFile (imagery_upload_data, f) {
  const form = new FormData()
  Object.keys(imagery_upload_data.fields).forEach(field => form.append(field, imagery_upload_data.fields[field]))
  const key = imagery_upload_data.key_prefix + f.name
  form.append('key', key)
  form.append('Content-Type', 'image/jpeg')
  form.append('file', f)

  return fetch(imagery_upload_data.url, { method: 'POST', body: form })
    .then(response => {
      if (!response.ok) {
        throw new Error(`Unexpected response HTTP ${response.status} ${response.statusText}`)
      }
    })
}
```

> Example JavaScript function to post an array of files using the `uploadFile`
> function above, using `imagery_upload_data`
> retrieved from the example above:

```js
function uploadFiles (imagery_upload_data, files) {
  return uploadNext()

  function uploadNext () {
    return new Promise(function (resolve, reject) {
      const file = files.shift()
      if (file) {
        uploadFile(imagery_upload_data, file)
          .then(uploadNext)
          .catch(reject)
      } else {
        resolve()
      }
    })
  }
}
```


The required parameters are included in the `imagery_upload_data` object: this object has a `url` property indicating the URL to POST imagery to, and a `fields` object, listing the HTTP form data fields required for the POST request. In addition to these fields, the form must also include a `key` field, this is the destination path for the uploaded file: the key must start with specified `key_prefix` and its value must be unique for each image, usually the key is set from the `key_prefix` and the uploaded file's filename. Note that subdirectories are not allowed and the filename can't contain special characters like `/`, `\` and `:`.

### HTTP Request

`POST https://solvi.ag/api/v1/projects/<project_id>/begin_upload`

### Parameters

Parameter |  | Description
--------- | ------- | -----------
project_id | required | Project ID given when project is created

## Upload remote imagery

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://solvi.ag/api/v1/projects/<project_id>/process_external_ortho"
  -d '{
    "ortho_url": "https://example.com/my_ortho.tiff"
  }'
```

> Example response:

```json
{
  "status": "ok"
}
```

This will start processing a prestitched ortho photo located at the URL provided in the `ortho_url` parameter. Note that the ortho must be possible to download through a simple HTTP GET request without any other authentication than is present in the URL itself. For example a presigned S3 URL is suitable.

You should _not_ use the `complete_upload` endpoint in combination with this endpoint, processing will start immediately when using this endpoint.

Note that this operation is only valid for projects of `prestitched` type.

### HTTP Request

`POST https://solvi.ag/api/v1/projects/<project_id>/process_external_ortho`

### Parameters

Parameter |  | Description
--------- | ------- | -----------
project_id | required | Project ID given when project is created
ortho_url  | required | The URL to download the prestitched ortho from

## Processing uploaded imagery

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://solvi.ag/api/v1/projects/<project_id>/complete_upload"
```

> Example response:

```json
{
  "status": "ok",
}
```

When project imagery has been uploaded, the upload must be completed, which will start the processing of the imagery.

If processing was successfully started, the HTTP response code will be `200`. In addition, a response of `402` (Payment required) can also be returned, if the upload or credit quota for the user is not sufficient for processing the upload.

### HTTP Request

`POST https://solvi.ag/api/v1/projects/<project_id>/complete_upload`

### Parameters

Parameter |  | Description
--------- | ------- | -----------
project_id | required | Project ID given when project is created

## Get projects

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  'https://solvi.ag/api/v1/projects?field_geom={"type":"FeatureCollection","features":[{"type":"Feature","geometry":{"type":"Polygon","coordinates":[[[100.0, 0.0],[101.0, 0.0],[101.0, 1.0],[100.0, 1.0],[100.0, 0.0]]]}]}'
```

> Example response:

```json
  [
    {
      "project_id": 9999,
      "status": "processed",
      "name": "25-JUN-2018",
      "plant_counts": "published",
      "field": {
          "id": 9999,
          "name": "Winter Wheat",
          "identfier": "WW-01",
          "farm": {
              "id": 656,
              "name": "Borgeby Farm"
          }
      },
      "survey_date": "2018-06-25T19:19:27.000Z",
      "upload_date": "2018-06-25T20:50:14.870Z",
      "url": "https://solvi.ag/projects/9999",
      "thumbnail_url": "https://solvi.ag/projects/9999/thumbnail.png",
      "metadata": null
    },
    {
      "name": "New project",
      "url": "https://solvi.ag/projects/new",
      "metadata": null
    }
  ]
```

This endpoint retrieves all projects created by the user or shared with user by others. If field boundaries are provided, only projects whose extent overlaps boundaries are returned.

### HTTP Request

`GET https://solvi.ag/api/v1/projects`

### Parameters

Parameter | | Description
--------- | ----------- | -----------
field_geom | optional | Boundaries of the field as a polygon in [GeoJSON format](https://geojson.org/geojson-spec.html#introduction) and EPSG:4326 coordinate system(lonlat).

## Project outputs

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  'https://solvi.ag/api/v1/projects/<project-id>
```

> Example response:

```json
  {
    "project_id": 9999,
    "status": "processed",
    "name": "25-JUN-2018",
    "plant_counts": "published",
    "field": {
        "id": 9999,
        "name": "Winter Wheat",
        "identfier": "WW-01",
        "farm": {
            "id": 656,
            "name": "Borgeby Farm"
        }
    },
    "survey_date": "2018-06-25T19:19:27.000Z",
    "upload_date": "2018-06-25T20:50:14.870Z",
    "url": "https://solvi.ag/projects/9999",
    "thumbnail_url": "https://solvi.ag/projects/9999/thumbnail.png",
    "resources": {
      "thumbnail": "https://solvi.ag/projects/9999/thumbnail.png",
      "ortho": "https://solvi-projects.s3.eu-west-1.amazonaws.com/uploads/ed12e5f6-b6c9-4df8-9522-1dbc29be854b/results/ortho.tiff?...",
      "dem": "https://solvi-projects-dev.s3.eu-west-1.amazonaws.com/uploads/ed12e5f6-b6c9-4df8-9522-1dbc29be854b/results/dem.tiff?..."
    },
    "metadata": null
  },
```

This endpoint retrieves projects created by the user or shared with the user by others.

The included `resources` are URLs that can be used to fetch the project outputs. These resource URLs are temporary, with a lifetime of 15 minutes before they expire.

### HTTP Request

`GET https://solvi.ag/api/v1/projects/<project_id>`

### Parameters

Parameter |  | Description
--------- | ------- | -----------
project_id | required | Project ID given when project is created

## Project tiled imagery

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  'https://solvi.ag/api/v1/projects/<project-id>/tiles/<type>
```

> Example response:

```json
{
  "url": "https://ts2.solvi.nu/.../results/ortho.tiff/rgb/rgb/tile/{z}/{x}/{y}.png?token=...",
  "type": "tileset/orthomosaic",
  "resolution": 0.15961538646165616,
  "bounds": {
    "value": [
      -115.407127627406,
      33.13991737241373,
      -115.39728408213948,
      33.144345906954
    ],
    "crs": "EPSG:4326"
  }
}
```

This endpoint retrieves a tile template URL, suitable for use with popular map clients like [OpenLayers](https://openlayers.org/) or [Leaflet](https://leafletjs.com/). The URLs are temporary, with a lifetime of at least 48 hours.

Note that all types are not necessarily available for all projects. Requesting a tile type that is not available will give a HTTP 404 response.

For `ortho` and `dem` tiles (when response `type` is `"tileset/orthomosaic`), geographic bounds are returned in the `bounds` property, and the native
resolution (meters/pixel) is given by `resolution`.

For `plant_counts` (when response `type` is `"tileset/plant_counts`), `number_plants` and `number_missing` give totals for the detected plants.

### HTTP Request

`GET https://solvi.ag/api/v1/projects/<project-id>/tiles/<type>`

### Parameters

Parameter |  | Description
--------- | ------- | -----------
project_id | required | Project ID given when project is created
type       | required | Type of tiles to fetch; currently one of `ortho` for RGB orthomosaic, `dem` for colored elevation model or `plant_counts` for plant counts

## Webhooks

Webhooks allow setting up integrations that subscribe to certain events from Solvi. When one of those events is triggered, an HTTP POST payload is sent to the webhook's configured URL. Currently, project status (event type `status_changed`) and plant counts published (event type `plant_counts_published`) are the three available events in Solvi.

> Example webhook request payload

```json
{
  "project_id": 8993,
  "event_type": "status_changed",
  "old_status": "not_processed",
  "new_status": "processed"
}
```

The webhook is configured when [creating a project](#create-project) or when [creating a plant count](#create-plant-counts) by specifing the `webhook` parameter, which should contain the URL of the webhook.

### Webhook events

The type of event that occured is determined by the `event_type` parameter of the request body. Currently, there are two supported event types: 

* `status_changed`, indicates that the project's status has changed
* `plant_counts_published`, triggered when plant counts have been published for the project

#### Status changes

The status change messages contains `old_status`, the status of the project before this change, as well as `new_status`, the project's updated status.

There are three different project statuses:

* `not_processed` - the project has been created but not yet processed, project outputs will not be available
* `processed` - the project has been processed and its outputs are available
* `failed` - the processing for this project failed, outputs are not available

#### Plant Counts Published

Notifies that plant counts data has been published for this project. The results can be fetched using the tiles endpoint with `plant_counts` as type.

### Securing webhooks

To ensure that Solvi is the sender of the webhook requests, you can optionally also specify a secret token when registering the webhook, by using the `webhook_secret` parameter: the secret can be any string of your choosing.

> Example signature header

```
X-Solvi-Signature: sha1=494e5dbdd1afbe4d44091bf86872b5eb4b9133e5
```

When a secret token has been specified, Solvi will include the HTTP header `X-Solvi-Signature`, which will contain an HMAC-SHA1 signature of the body.

This follows the same pattern as [securing webhooks on GitHub](https://docs.github.com/en/developers/webhooks-and-events/securing-your-webhooks), except for using the header `X-Solvi-Signature` instead.

# Plant Counts

The plant counts API is under development and currently only supports a limited number of use cases:

* Automatic plant counts in scouting projects
* Request plant counts done for you (by Solvi personel) in any type of project

## Create Plant Counts

> Example request:

```shell
curl 
  -X POST 
  -H "Authorization: Bearer <user-jwt-token>" 
  -H 'Content-Type: application/json'
  -d '{"model": "base-models/shape"}'
  https://solvi.ag/api/v1/projects/<project-id>/plant_counts
```

> Example response:

```json
  {
    "status": "Created",
    "job_id": "88a2af8601cf2b7c4e14979f8df73bca7352ccce",
  }
```

This initiates a plant count for the specified project. A plant count can either be:

* automatic, meaning Plant AI will run unsupervised
* detection done for you by Solvi personel, supervising the detection to ensure high quality.

When using the API, automatic plant counts are currently only supported for scouting projects.

To create a supervised "done for you" plant count, set the `request_done_for_you` parameter to `true`.

Creating a plant count is an asynchronous process, after creation the plant count will be processing, and the results can not be accessed until it has completed. This asynchronous process is called a _job_. To check the job's status, it can either be polled (see below) or a webhook URL can be submitted when starting the plant count; this webhook will be called when the job's status changes. Plant count webhooks only work in automatic mode, "done for you" plant counts can either use polling, or add a project webhook and listen to when the project's plant count is published. See [webhooks](#webhooks) for more details. Note that "done for you" type plant counts can take up
to 24 hours to complete.

The detection is performed using a detection model. By default, two different models are available:

* `base-models/point` - used for basic counts and is applicable to crops like corn or vegetables in early growth-stages
* `base-models/shape` - includes size estimates and health for individual plants, applicable for pre-harvest crops or trees

In addition, more models might be available on a per-user basis.

### HTTP Request

`POST https://solvi.ag/api/v1/projects/<project-id>/plant_counts`

### Parameters

Parameter | | Description
--------- | ----------- | -----------
model     | required | The detection model to be used
detect_rows | optional | `true` if row detection should be performed; only supported in automatic scouting plant counts
request_done_for_you | optional | `true` if detection should be done for you by Solvi personel
webhook   | optional | URL of the webhook to send status updates to

## Plant Count data

> Example request:

```shell
curl 
  -X GET
  -H "Authorization: Bearer <user-jwt-token>" 
  https://solvi.ag/api/v1/projects/<project-id>/plant_counts/<job-id>
```

> Example response:

```json
  {
    "status": "completed",
    "results": [
      "https://solvi-staging-production.s3.eu-west-1.amazonaws.com/uploads/..."
    ]
  }
```

Gets the status of a previously started plant count, or overview information about a processed plant count.

Depending on status of the detection job, different HTTP status codes will be used for the response:

* `200`: completed successfully and the body contains JSON information about the result
* `202`: processing is not yet complete
* `404`: unknown job id or the job's result has expired and been removed
* `500`: an internal error occurred during processing

When processing has completed, the response will contain a `results` property which lists URLs that can be used to get the detection results. These URLs are valid for at least 24 hours. For scouting projects, a detection results in a JSON file containing a summary of the detection, as well as one GeoJSON file per image in the project.

The results of a detection are kept in storage for one week. After this time, the results are removed from storage and requesting the results will return a 404 code. If a detection is published, the results will be kept in storage.

Note that "done for you" plant counts in legacy projects, processed before 2023-01-01, the returned result _may_
be a single ZIP archive file containing a Shapefile. In newer projects, the returned data will always be uncompressed data in multiple files.

## Plant Count Webhook

In addition to polling a plant count for data, you can also register a webhook that will be notified when the plant count has been processed. The webhook URL is specified when you [create the plant count](#create-plant-counts). The webhook will [sign requests using the project's webhook secret](#securing-webhooks) if it is set.

The webhook will currently only receive a single type of event: `job_status_changed`. This event triggers when the plant detection job's status changes. It contains:

* `job_id` for the detection that changed
* `project_id` of the project it refers to
* `old_status` the status of the project before this change
* `new_status` the job's updated status.

There are three different statuses:

* `processing` - the plant count is being processed
* `completed` - the plant count has been processed succesfully and its results are available
* `fail` - the detection failed



## Publishing Plant Counts

> Example request:

```shell
curl 
  -X POST
  -H "Authorization: Bearer <user-jwt-token>" 
  https://solvi.ag/api/v1/projects/<project-id>/plant_counts/<job-id>/publish
```

> Example response:

```json
  {
    "status": "Success",
  }
```

After a plant count job has completed succesfully, its results can be published. This will achieve two things:

* The results can be viewed from the Plant Counts tool in Solvi's web app
* Results will not be removed from storage after a week

## Listing a Project's Plant Counts

> Example request:

```shell
curl 
  -X GET
  -H "Authorization: Bearer <user-jwt-token>" 
  https://solvi.ag/api/v1/projects/<project-id>/plant_counts
```

> Example response:

```json
  [
    {
      "id": "88a2af8601cf2b7c4e14979f8df73bca7352ccce",
      "status": "completed",
      "created_at": "2019-02-08T09:18:37.655Z",
    },
    {
      "id": "34a2af8601cf2b7c4e14979f8df73bca7352aef43",
      "status": "published",
      "created_at": "2021-03-12T14:33:31.241Z",
    }
  ]
```

Lists all created plant counts for a project with their id and status. Status is one of:

* `processing`: the detection has not yet completed
* `completed`: detection has finished and can be viewed
* `published`: this detection is the one currently published for the project
* `fail`: detection failed

## Deleting a Plant Count

> Example request:

```shell
curl 
  -X DELETE
  -H "Authorization: Bearer <user-jwt-token>" 
  https://solvi.ag/api/v1/projects/<project-id>/plant_counts/<job-id>
```

> Example response:

```json
  {
    "status": "deleted",
  }
```

Removes the results of a plant count. Please note that "done for you" plant counts can't be deleted.

# Zonal Statistics

The Zonal Statistics API lets you create zones in a project and create various statistics for them: primarily various vegetation indices as well as plant count statistics for projects where plant counts have been published.

## Creating and calculating statistics for zones

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>" 
  -H "Content-Type: application/json"
  -d '{
    "indices": ["vari"],
    "extra_data": {"plant_counts": true},
    "geojson": {
      "type": "FeatureCollection",
      "features": [
        {
          "type": "Feature",
          "properties": {},
          "geometry": {
            "type": "Polygon"
            "coordinates": [
              [
                [
                  -115.40068410376747,
                  33.14301501895726
                ],
                [
                  -115.40068410376747,
                  33.142561119287876
                ],
                [
                  -115.4001781509446,
                  33.142561119287876
                ],
                [
                  -115.4001781509446,
                  33.14301501895726
                ],
                [
                  -115.40068410376747,
                  33.14301501895726
                ]
              ]
            ],
          }
        }
      ]
    }
  }'
  https://solvi.ag/api/v1/projects/<project-id>/zonal_statistics'
```

> Example response:

```json
{
    "type": "FeatureCollection",
    "features": [
        {
            "type": "Feature",
            "properties": {
                "vari_p25": -0.12272727489471436,
                "vari_p75": -0.051350148394703865,
                "vari_mean": -0.08275701161977407,
                "vari_median": -0.11009174585342407,
                "vari_stdev": 0.05680577057554646,
                "vari_min": -0.1606217622756958,
                "vari_max": 0.1172839477658
                [...]
```

Sets the zones for a project and calculates the provided statistics. The zones should be provided through the `geojson` parameter: it should contain a GeoJSON FeatureCollection with Polygon features. Other GeoJSON objects or invalid GeoJSON will result in a `400 Bad request` response.

In addition to the zone features, the `indices` parameter lists the vegetation indices that should be calculated for the zones. The indices should be listed as lowercase abbreviations, for example VARI index should be listed as `"vari"`. Note that the available indices vary depending on the project's bands: for example, NDVI is only available if the project's orthophoto has red and near-infrared (NIR) bands. Listing unknown indices or indices that can't be calculated will also result in a `400 Bad request` response.

For projects with published plant counts, plant count statistics can also be calculated by the optional parameter `extra_data`, which in this case should be set to `{"plant_counts": true}`.

The endpoint responds with a GeoJSON FeatureCollection with the same geometries that were provided, but with properties added for the calculated statistics. This response is also saved with the project, and can be viewed on Solvi's web site, and also later fetched with the GET endpoint below.

### HTTP Request

`POST https://solvi.ag/api/v1/projects/<project-id>/zonal_statistics`

### Parameters

Parameter   |             | Description
----------- | ----------- | -----------
indices     | required    | List in vegetation indices to calculate; index names should be lower case strings
geojson     | required    | GeoJSON FeatureCollection representing the zone features
extra_data  | optional    | Extra data layers to calculate statistics for, expressed as an object with string layer names as keys and boolean values to indicate if layer should be used

## Getting Zonal Statistics for a project

> Example request:

```shell
curl 
  -X GET
  -H "Authorization: Bearer <user-jwt-token>" 
  https://solvi.ag/api/v1/projects/<project-id>/zonal_statistics
```

> Example response:

```json
{
    "type": "FeatureCollection",
    "features": [
        {
            "type": "Feature",
            "properties": {
                "vari_p25": -0.12272727489471436,
                "vari_p75": -0.051350148394703865,
                "vari_mean": -0.08275701161977407,
                "vari_median": -0.11009174585342407,
                "vari_stdev": 0.05680577057554646,
                "vari_min": -0.1606217622756958,
                "vari_max": 0.1172839477658
                [...]
```

Fetches any saved zonal statistics for a project. The zones are returned as a GeoJSON FeatureCollection with statistics in each feature's `properties`. If no zones are defined for this project, a `404 Not found` is returned.
