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

## HTTP Response Codes

Like any REST API, Solvi's API indicates the result of a request through the HTTP status code in the response. After sending a request to an endpoint, the returned status must be checked to ensure the request succeeded.

- _Successful_ results return a `2xx` status:
  - `200` is used for `GET`, `PUT` and `DELETE` requests
  - `201` is returned after successfully creating a new entity from a `POST` request
  - `202` is returned when a `POST` request has started an asynchronous [task](#tasks); the response's `Location` header is the URL of the task
- An _invalid request_ is indictaed by a `4xx` status:
  - `400` is used to indicate the parameters were invalid
  - `404` if the requested resource does not exist, or the user does not have access to it
  - `402` if the account does not have enough quota
  - `403` if the authenticated user is not allowed to perform this request
  - `409` if the request conflicts with the resource's current state, for example deleting a field that still has projects
  - `429` if the user has too many requests or tasks running at the same time; the `Retry-After` header gives the number of seconds to wait before trying again
- `302` is used if the request was not properly authenticated (this will likely change in the future)
- `5xx` means an internal error occurred: please contact support since this indicates a

> Example error response:

```json
{
  "status": "not found",
  "message": "not found"
}
```

Error responses have a JSON body with a `status` property and, in most cases, a `message` describing the error.

## Lists and paging

> Example request for the first page of 100 projects:

```shell
curl -i -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  "https://api.solvi.ag/v1/projects?limit=100"
```

> Response headers when more projects follow:

```
Link: <https://api.solvi.ag/v1/projects?continuation_token=MTIzNA%3D%3D&limit=100>; rel="next"
```

Endpoints that list farms, fields, projects or users return the records ordered by id, oldest first, and return at most 1000 records per request. When more records follow, the response includes a `Link` header with `rel="next"`, containing the URL of the next page. Continue requesting the URL from the `Link` header until a response has no such header. The response body is always a JSON array of the records.

All list endpoints accept these parameters, in addition to the ones listed for each endpoint:

| Parameter          |          | Description                                                                                                                         |
| ------------------ | -------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| limit              | optional | The maximum number of records to return, 1-1000; default is 1000                                                                   |
| continuation_token | optional | Continue a list after the previous page; this is set in the `Link` header's URL and should not be constructed by the client         |
| search             | optional | Only return records whose name contains this text, case insensitive; see each endpoint for which names are searched                 |
| created_from       | optional | Only return records created on or after this date, formatted as YYYY-MM-DD                                                          |
| created_to         | optional | Only return records created on or before this date, formatted as YYYY-MM-DD                                                         |

The farm, field and project lists also accept:

| Parameter |          | Description                                                                                                                                                                       |
| --------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| scope     | optional | `all` (the default) returns everything the user has access to, including records of other members of the user's organisation and farms shared with the user; `own` returns only records the user owns |
| user_id   | optional | Only return records owned by the user with this id                                                                                                                               |

Dates that are not formatted as YYYY-MM-DD, and other invalid parameter values, give a `400` response.

# Authentication

> Example API key request:

```shell
curl "api_endpoint_here"
  -H "X-Api-Key: <your-api-key>"
```

> Make sure to replace `<your-api-key>` with your API key.

To use Solvi API you first need an _API key_. You will get an API key from Solvi, [contact us](mailto:support@solvi.nu) to get your key. If you for some
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

Instead of using the API key, requests that act on behalf of a specific user, like creating or getting projects, should use a user-specific token to authenticate as the user. When using a user-specific token, _do not_ include the API key in the same request. A user-specific token has a limited lifetime and is less security sensitive compared to the API key. For example, a user token can be sent to and used directly from for example a browser, or for passwordless authentication when the user is redirected from your portal to Solvi. A user-specific token can be used for a limited amount of time (currently 24 hours) before it expires. A user-specific token is created by using the [endpoint to create a user-specific token](#generate-user-specific-token).

User specific requests should use the `Authorization` header _instead of_ the `X-Api-Key` header:

`Authorization: Bearer <user-specific-token>`

# Users

## Register new user

> Example request:

```shell
curl -X POST
  -H "X-Api-Key: <your-api-key>"
  -H "Content-Type: application/json"
  -d '{ "user": { "email": "john@example.com", "password": "password", "first_name": "John", "last_name": "Doe"}}'
  "https://api.solvi.ag/v1/users"
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

`POST https://api.solvi.ag/v1/users`

### Parameters

| Parameter  |          | Description |
| ---------- | -------- | ----------- |
| email      | required | Email       |
| password   | required | Password    |
| first_name | required | First name  |
| last_name  | required | Last name   |
| company    | optional | Company     |

<aside>
Parameters above must be wrapped into `user` attribute and sent as JSON payload in POST request, see the example.
</aside>

If a user with the email already exists, the response is `409`.

## Get users

> Example request:

```shell
curl -X GET
  -H "X-Api-Key: <your-api-key>"
  "https://api.solvi.ag/v1/users?search=doe"
```

> Example response:

```json
[
  {
    "id": 182,
    "email": "john@example.com",
    "first_name": "John",
    "last_name": "Doe",
    "company": "Doe Farming",
    "created_at": "2024-03-12T08:41:19.112Z",
    "locked": false
  }
]
```

Gets the users registered through your API key. The list is [paged](#lists-and-paging).

### HTTP Request

`GET https://api.solvi.ag/v1/users`

### Parameters

| Parameter |          | Description                                                                 |
| --------- | -------- | --------------------------------------------------------------------------- |
| search    | optional | Only return users whose name, email or company contains this text           |
| email     | optional | Only return the user with this email; case insensitive                      |

The list also accepts the parameters common to [all lists](#lists-and-paging).

## Get user

> Example request:

```shell
curl -X GET
  -H "X-Api-Key: <your-api-key>"
  "https://api.solvi.ag/v1/users/<user_id>"
```

> Example response:

```json
{
  "id": 182,
  "email": "john@example.com",
  "first_name": "John",
  "last_name": "Doe",
  "company": "Doe Farming",
  "created_at": "2024-03-12T08:41:19.112Z",
  "locked": false
}
```

Gets a single user registered through your API key.

### HTTP Request

`GET https://api.solvi.ag/v1/users/<user_id>`

## Update user

> Example request:

```shell
curl -X PUT
  -H "X-Api-Key: <your-api-key>"
  -H "Content-Type: application/json"
  -d '{ "user": { "company": "Doe Farming AB" }}'
  "https://api.solvi.ag/v1/users/<user_id>"
```

> Example response:

```json
{
  "status": "ok"
}
```

Updates a user registered through your API key. Only the parameters included in the request are changed.

### HTTP Request

`PUT https://api.solvi.ag/v1/users/<user_id>`

### Parameters

| Parameter  |          | Description |
| ---------- | -------- | ----------- |
| email      | optional | Email       |
| password   | optional | Password    |
| first_name | optional | First name  |
| last_name  | optional | Last name   |
| company    | optional | Company     |

<aside>
Parameters above must be wrapped into `user` attribute, as when registering a user.
</aside>

## Generate user-specific token

> Example request:

```shell
curl -X GET
  -H "X-Api-Key: <your-api-key>"
  "https://api.solvi.ag/v1/users/<user_id>/token"
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

`GET https://api.solvi.ag/v1/users/<user_id>/token`

### Parameters

| Parameter |          | Description                        |
| --------- | -------- | ---------------------------------- |
| user_id   | required | User ID given when user is created |

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
  "https://api.solvi.ag/v1/farms"
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

`POST https://api.solvi.ag/v1/farms`

### Parameters

| Parameter |          | Description                 |
| --------- | -------- | --------------------------- |
| name      | required | Name of the the farm        |
| metadata  | optional | Optional arbitrary metadata |

## Get farms

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://api.solvi.ag/v1/farms"
```

> Example response:

```json
[
  {
    "id": 656,
    "name": "My farm",
    "user_id": 182,
    "archived": false,
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

Gets the farms the user has access to and a list of fields related to each farm. This includes farms shared with the user and, for members of an organisation, the organisation's farms the user has access to. The list is [paged](#lists-and-paging).

### HTTP Request

`GET https://api.solvi.ag/v1/farms`

### Parameters

| Parameter |          | Description                                                                              |
| --------- | -------- | ---------------------------------------------------------------------------------------- |
| search    | optional | Only return farms whose name contains this text                                          |
| archived  | optional | `true` to return only archived farms, `false` to return only active farms; default both |

The list also accepts the parameters common to [all lists](#lists-and-paging), including `scope` and `user_id`.

## Get farm

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  "https://api.solvi.ag/v1/farms/<farm_id>"
```

Gets a single farm, in the same format as an item of the [farms list](#get-farms).

### HTTP Request

`GET https://api.solvi.ag/v1/farms/<farm_id>`

## Update farm

> Example request:

```shell
curl -X PUT
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{"farm": {"name": "Home farm", "archived": true}}'
  "https://api.solvi.ag/v1/farms/<farm_id>"
```

> Example response:

```json
{
  "status": "ok"
}
```

Updates a farm. Only the parameters included in the request are changed. The user must be allowed to edit the farm, otherwise the response is `403`.

### HTTP Request

`PUT https://api.solvi.ag/v1/farms/<farm_id>`

### Parameters

| Parameter |          | Description                                           |
| --------- | -------- | ----------------------------------------------------- |
| name      | optional | Name of the farm                                      |
| archived  | optional | `true` to archive the farm, `false` to make it active |
| metadata  | optional | Optional arbitrary metadata                           |

## Delete farm

> Example request:

```shell
curl -X DELETE
  -H "Authorization: Bearer <user-jwt-token>"
  "https://api.solvi.ag/v1/farms/<farm_id>"
```

> Example response:

```json
{
  "status": "ok"
}
```

Deletes a farm and its fields. The user must be allowed to edit the farm, otherwise the response is `403`.

A farm can only be deleted when all projects on its fields have been [deleted](#delete-project), and a user must always keep at least one farm. Otherwise, the response is `409`.

### HTTP Request

`DELETE https://api.solvi.ag/v1/farms/<farm_id>`

# Fields

## Create field

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{"field": { "name": "Winter Wheat", "geom": "{\"type\":\"Polygon\",\"coordinates\":[[[100.0, 0.0],[101.0, 0.0],[101.0, 1.0],[100.0, 1.0],[100.0, 0.0]]]}"}}'
  "https://api.solvi.ag/v1/fields"
```

> Example response:

```json
{
  "status": "success",
  "field_id": 793
}
```

Creates a new field. Field boundaries can be provided as Polygon or MultiPolygon in GeoJSON format. The farm to organize the field under can also be provided; the user must be allowed to edit that farm, otherwise the response is `400`. The response contains `field_id` which can be later used to relate projects to the specific field.

### HTTP Request

`POST https://api.solvi.ag/v1/fields`

### Parameters

| Parameter |          | Description                                                                                                                                                          |
| --------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| name      | required | Name of the the field                                                                                                                                                |
| geom      | optional | Boundaries of the field as a Polygon or Multipolygon in [GeoJSON format](https://geojson.org/geojson-spec.html#introduction) and EPSG:4326 coordinate system(lonlat) |
| farm_id   | optional | The id of the farm to put the field under; if not specified, the farm the user created last is used                                                                  |
| metadata  | optional | Optional arbitrary metadata                                                                                                                                          |

## Get fields

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://api.solvi.ag/v1/fields"
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
        "url": "https://app.solvi.ag/projects/1291",
        "thumbnail_url": "https://app.solvi.ag/projects/1291/thumbnail.png",
        "metadata": null
      }
    ],
    "farm": {
      "id": 656,
      "name": "My farm",
      "metadata": null
    },
    "geom": {
      "type": "Polygon",
      "coordinates": [[[100.0, 0.0], [101.0, 0.0], [101.0, 1.0], [100.0, 1.0], [100.0, 0.0]]]
    },
    "metadata": null
  }
]
```

Gets the fields on the farms the user has access to, and a list of the processed projects related to each field. The list is [paged](#lists-and-paging).

### HTTP Request

`GET https://api.solvi.ag/v1/fields`

### Parameters

| Parameter |          | Description                                      |
| --------- | -------- | ------------------------------------------------ |
| farm_id   | optional | Only return fields on the farm with this id      |
| search    | optional | Only return fields whose name contains this text |

The list also accepts the parameters common to [all lists](#lists-and-paging), including `scope` and `user_id`; these refer to the owner of the field's farm.

## Get field

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  "https://api.solvi.ag/v1/fields/<field_id>"
```

Gets a single field, in the same format as an item of the [fields list](#get-fields).

### HTTP Request

`GET https://api.solvi.ag/v1/fields/<field_id>`

## Update field

> Example request:

```shell
curl -X PUT
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{"field": {"name": "Spring Wheat"}}'
  "https://api.solvi.ag/v1/fields/<field_id>"
```

> Example response:

```json
{
  "status": "ok"
}
```

Updates a field. Only the parameters included in the request are changed. The user must be allowed to edit the field's farm, otherwise the response is `403`. A field can be moved to another farm by setting `farm_id`; the user must be allowed to edit that farm too, otherwise the response is `400`.

### HTTP Request

`PUT https://api.solvi.ag/v1/fields/<field_id>`

### Parameters

| Parameter |          | Description                                                                                                                                                          |
| --------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| name      | optional | Name of the the field                                                                                                                                                |
| geom      | optional | Boundaries of the field as a Polygon or Multipolygon in [GeoJSON format](https://geojson.org/geojson-spec.html#introduction) and EPSG:4326 coordinate system(lonlat) |
| farm_id   | optional | The id of the farm to move the field to                                                                                                                              |
| metadata  | optional | Optional arbitrary metadata                                                                                                                                          |

## Delete field

> Example request:

```shell
curl -X DELETE
  -H "Authorization: Bearer <user-jwt-token>"
  "https://api.solvi.ag/v1/fields/<field_id>"
```

> Example response:

```json
{
  "status": "ok"
}
```

Deletes a field. The user must be allowed to edit the field's farm, otherwise the response is `403`. A field can only be deleted when all its projects have been [deleted](#delete-project), otherwise the response is `409`.

### HTTP Request

`DELETE https://api.solvi.ag/v1/fields/<field_id>`

# Projects

## Create project

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{ "type": "overlapping", "field_id": "field_1", "field_name": "Wheat Field", "field_geom": "{\"type\":\"Polygon\",\"coordinates\":[[[100.0, 0.0],[101.0, 0.0],[101.0, 1.0],[100.0, 1.0],[100.0, 0.0]]]}" }'
  "https://api.solvi.ag/v1/projects"
```

> Example response:

```json
  {
    "status": "success",
    "project_id": 142,
    "project_url": "https://app.solvi.ag/projects/142/photos/upload",
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

There are two _types_ of projects: `overlapping` (the default) and `stitched`:

- `overlapping` should be used if you have a number of photos taken which should be stitched into a map by Solvi; this is the default and what you normally use
- `stitched` can be used if you already have a single [GeoTIFF](https://en.wikipedia.org/wiki/GeoTIFF) that have been stitched by another system

Projects can be connected to a Field. When multiple projects are related to the same Field, they appear in the same map view when data is processed. This allows for easier navigation between imagery over the same Field and over the time data comparison.

Fields can be either be:

- [created beforehand](#create-field); for this case, set the `field_id` parameter to the previously created field's `id`; the user must be allowed to edit the field's farm, or the farm must be shared with the user, otherwise the response is `403`
- created on the fly by sending `field_name` and `field_geom` (optional), which will create a new field for the project on the farm the user created last; do not include `field_id`

Optionally, a project can be created with a so-called _webhook_ that will be called every time the status of the project changes. This makes it possible for integration to for example react when a project finishes processing, without having to use polling to check the project's status. To add a webhook, specify the URL to be called with the `status_webhook` parameter. See the section on [webhooks](#webhooks) for details.

The project's crop, survey date and altitude, data which will be displayed in Solvi's UI, can also be submitted.

### HTTP Request

`POST https://api.solvi.ag/v1/projects`

### Parameters

| Parameter      |          | Description                                                                                                                                          |
| -------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| type           | optional | The type of imagery for this project: `overlapping` or `stitched`; default is `overlapping`                                                          |
| field_id       |          | Solvi field id as returned from [Fields API](#fields)                                                                                                |
| field_name     |          | Name of the the field                                                                                                                                |
| field_geom     | optional | Boundaries of the field as a polygon in [GeoJSON format](https://geojson.org/geojson-spec.html#introduction) and EPSG:4326 coordinate system(lonlat) |
| status_webhook | optional | A URL to be called when the project's status changes                                                                                                 |
| webhook_secret | optional | A token to use to sign webhook requests, see [webhooks](#webhooks) for details                                                                       |
| survey_date    | optional | Date survey was conducted, formatted as YYYY-MM-DD                                                                                                   |
| crop           | optional | The documented crop; free form string                                                                                                                |
| altitude       | optional | Altitude the imagery was taken from in meters                                                                                                        |
| metadata       | optional | Optional arbitrary metadata                                                                                                                          |

## Update project data

> Example request:

```shell
curl -X PUT
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{
    "survey_date": "2022-08-18"
  }'
  "https://api.solvi.ag/v1/projects/<project_id>"
```

> Example response:

```json
{
  "status": "ok"
}
```

### HTTP Request

`PUT https://api.solvi.ag/v1/projects/<project_id>`

### Parameters

| Parameter   |          | Description                                        |
| ----------- | -------- | -------------------------------------------------- |
| survey_date | optional | Date survey was conducted, formatted as YYYY-MM-DD |
| crop        | optional | The documented crop; free form string              |
| altitude    | optional | Altitude the imagery was taken from in meters      |
| metadata    | optional | Optional arbitrary metadata                        |

This endpoint allows updating survey date, crop, altitude and metadata for a project, whatever its status. Only the parameters included in the request are changed. If the user is not allowed to edit the project, the response is `404`.

## Delete project

> Example request:

```shell
curl -X DELETE
  -H "Authorization: Bearer <user-jwt-token>"
  "https://api.solvi.ag/v1/projects/<project_id>"
```

> Example response:

```json
{
  "status": "ok"
}
```

Deletes a project. The project is no longer returned by the API or shown in Solvi. If the user is not allowed to edit the project, the response is `404`; if the user's account does not allow editing projects at all, the response is `403`.

### HTTP Request

`DELETE https://api.solvi.ag/v1/projects/<project_id>`

## Upload local imagery

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://api.solvi.ag/v1/projects/<project_id>/begin_upload"
```

> Example response:

```json
{
  "status": "ok",
  "imagery_upload_data": {
    "type": "post_form_data",
    "url": "https://solvi-production.s3.eu-west-1.amazonaws.com",
    "fields": {
      "x-amz-storage-class": "STANDARD_IA",
      "policy": "eyJleHBpcmF0aW9uIjoiMjAyNS0wNS0yMVQwNjowMjo1N1oiLCJjb25kaXRpb25zIjpbeyJidWNrZXQiOiJzb2x2aS1wcm9kdWN0aW9uIn0sWyJzdGFydHMtd2l0aCIsIiRrZXkiLCJ1cGxvYWRzLzgxMDQ0N2JkLWQ5MzEtNGJiNS05OTE0LTA2NGZlNDRiODUzZS9vcmlnaW5hbHMvIl0sWyJzdGFydHMtd2l0aCIsIiRDb250ZW50LVR5cGUiLCIiXSx7IngtYW16LXN0b3JhZ2UtY2xhc3MiOiJTVEFOREFSRF9JQSJ9LHsieC1gbXotY3JlZGVudGlhbCI6IkFLSUFVVk1IS0dUUUsyQVVBNExRLzIwMjUwNTE0L2V1LXdlc3QtMS9zMy9hd3M0X3JlcXVlc3QifSx7IngtYW16LWFsZ29yaXRobSI6IkFXUzQtSE1BQy1TSEEyNTYifSx7IngtYW16LWRhdGUiOiIyMDI1MDUxNFQwNjAyNTdaIn1dfQ==",
      "x-amz-credential": "AKIAUVMHKGTQK2AUA4LQ/20250514/eu-west-1/s3/aws4_request",
      "x-amz-algorithm": "AWS4-HMAC-SHA256",
      "x-amz-date": "20250514T060257Z",
      "x-amz-signature": "5811e53afc5d1f2e94fc614a29b699d89529d112eb432d7c6d332fafbbf9c987"
    },
    "key_prefix": "uploads/26c1ce11-c9bf-4825-821c-72e9f600a6cf/originals/",
    "number_parallel_hint": 3
  }
}
```

> or

```json
{
  "status": "ok",
  "imagery_upload_data": {
    "type": "url_template",
    "template": "https://uploads.solvi.ag/uploads/26c1ce11-c9bf-4825-821c-72e9f600a6cf/originals/$filename?token=eyJhbGciOiJIUzI1NiJ9.eyJrZXlQcmVmaXgiOiJ1cGxvYWRzLzVkM2UyZDIwLTIxYTMtNGMyOC1hZWYzLTIxYTU5Y2Y0MzhhZC9vcmlnaW5hbHMvIiwiZXhwIjoxNzQ3ODA3MjgwfQ.UQzDIM020OOrwsFwzEFCtIjLEJJiYYFsPZdwLqLZf58",
    "number_parallel_hint": 3
  }
}
```

After a project has been created, imagery can be uploaded to it by sending the `begin_upload` request. The response includes information required to upload imagery for the project. The response can be of two different types depending on your project type and customer configuration: `post_form_data` and `url_template`.

> Example JavaScript function to upload an image using `imagery_upload_data`
> retrieved from the example above:

```js
function uploadFile(imagery_upload_data, f) {
  let fetchPromise;
  switch (imagery_upload_data.type) {
    case "url_template" {
      const url = imagery_upload_data.template.replace("$filename", f.name);
      fetchPromise = fetch(url, {
        method: "PUT",
        body: f,
        headers: { "Content-Type": f.type },
      });
      break
    }
    case "post_form_data" {
      const form = new FormData();
      for (const key in imagery_upload_data.fields) {
        form.append(key, imagery_upload_data.fields[key]);
      }
      form.append("key", imagery_upload_data.key_prefix + f.name);
      form.append("Content-Type", f.type);
      form.append("file", f);

      fetchPromise = fetch(imagery_upload_data.url, {
        method: "POST",
        body: form,
      });
      break
    }
    default:
      throw new Error(`Unexpected upload type ${imagery_upload_data.type}`)
  }

  return fetchPromise.then((response) => {
    if (!response.ok) {
      throw new Error(
        `Unexpected response HTTP ${response.status} ${response.statusText}`
      );
    }
  });
}
```

> Example JavaScript function to post an array of files using the `uploadFile`
> function above, using `imagery_upload_data`
> retrieved from the example above:

```js
function uploadFiles(imagery_upload_data, files) {
  return uploadNext();

  function uploadNext() {
    return new Promise(function (resolve, reject) {
      const file = files.shift();
      if (file) {
        uploadFile(imagery_upload_data, file).then(uploadNext).catch(reject);
      } else {
        resolve();
      }
    });
  }
}
```

> Example Python implementation to upload a file using `imagery_upload_data` using the [requests](https://requests.readthedocs.io/en/latest/) library:

```python
import os
import requests

def upload_file(imagery_upload_data, file_path):
  with open(file_path, "rb") as f:
    if imagery_upload_data["type"] == "post_form_data":
      form_data = imagery_upload_data["fields"]
      key = imagery_upload_data["key_prefix"] + os.path.basename(file_path)
      form_data["key"] = key
      form_data["Content-Type"] = "image/jpeg"

      response = requests.post(
        imagery_upload_data["url"],
        data=form_data,
        files={"file": (key, f)},
      )
    elif imagery_upload_data["type"] == "url_template":
      url = imagery_upload_data["template"].replace(
        "$filename", os.path.basename(file_path)
      )
      response = requests.put(url, data=f)
    else:
      raise Exception(f'Unknown type "{imagery_upload_data["type"]}"')

    if response.status_code not in [200, 201, 204]:
      raise Exception(
        f"Error uploading image '{file_path}': HTTP {response.status_code} - {response.text}"
      )
```

### `post_form_data` method

The required parameters are included in the `imagery_upload_data` object: this object has a `url` property indicating the URL to POST imagery to, and a `fields` object, listing the HTTP form data fields required for the POST request. In addition to these fields, the form must also include a `key` field, this is the destination path for the uploaded file: the key must start with specified `key_prefix` and its value must be unique for each image, usually the key is set from the `key_prefix` and the uploaded file's filename. Note that subdirectories are not allowed and the filename can't contain special characters like `/`, `\` and `:`.

### `url_template` method

The response includes a property called `url_template` which should be used to create URLs to `PUT` files to. The template includes that part `$filename` which should be replaced with the actual name of the file being uploaded. Note that subdirectories are not allowed and the filename can't contain special characters like `/`, `\` and `:`.

### HTTP Request

`POST https://api.solvi.ag/v1/projects/<project_id>/begin_upload`

### Parameters

| Parameter  |          | Description                              |
| ---------- | -------- | ---------------------------------------- |
| project_id | required | Project ID given when project is created |

## Upload remote imagery

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://api.solvi.ag/v1/projects/<project_id>/process_external_ortho"
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

Note that this operation is only valid for projects of `stitched` type.

### HTTP Request

`POST https://api.solvi.ag/v1/projects/<project_id>/process_external_ortho`

### Parameters

| Parameter  |          | Description                                    |
| ---------- | -------- | ---------------------------------------------- |
| project_id | required | Project ID given when project is created       |
| ortho_url  | required | The URL to download the prestitched ortho from |

## Processing uploaded imagery

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  "https://api.solvi.ag/v1/projects/<project_id>/complete_upload"
```

> Example response:

```json
{
  "status": "ok"
}
```

When project imagery has been uploaded, the upload must be completed, which will start the processing of the imagery.

If processing was successfully started, the HTTP response code will be `200`. In addition, a response of `402` (Payment required) can also be returned, if the upload or credit quota for the user is not sufficient for processing the upload.

### HTTP Request

`POST https://api.solvi.ag/v1/projects/<project_id>/complete_upload`

### Parameters

| Parameter  |          | Description                              |
| ---------- | -------- | ---------------------------------------- |
| project_id | required | Project ID given when project is created |

## Get projects

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  'https://api.solvi.ag/v1/projects?field_geom={"type":"FeatureCollection","features":[{"type":"Feature","properties":{},"geometry":{"type":"Polygon","coordinates":[[[100.0, 0.0],[101.0, 0.0],[101.0, 1.0],[100.0, 1.0],[100.0, 0.0]]]}}]}'
```

> Example response:

```json
[
  {
    "project_id": 9999,
    "status": "processed",
    "name": "25-JUN-2018",
    "plant_counts": "published",
    "user": {
      "name": "Joe Smith",
      "email": "joe@example.com"
    },
    "plant_counts": "",
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
    "url": "https://app.solvi.ag/projects/9999",
    "thumbnail_url": "https://app.solvi.ag/projects/9999/thumbnail.png",
    "metadata": null,
    "number_images": 322,
    "bands": "rgb",
    "crop": "Wheat",
    "relative_altitude": 99.9,
    "absolute_altitude": 95.443,
    "extent": {
      "type": "MultiPolygon",
      "coordinates": [
        [
          [
            [12.335404361111113, 57.30721566666666],
            [12.337469722222222, 57.30721566666666],
            [12.337469722222222, 57.30810941666666],
            [12.335404361111113, 57.30810941666666],
            [12.335404361111113, 57.30721566666666]
          ]
        ]
      ]
    }
  },
  {
    "name": "New project",
    "url": "https://app.solvi.ag/projects/new",
    "metadata": null
  }
]
```

This endpoint retrieves the projects created by the user and the projects on farms the user has access to: farms shared with the user and, for members of an organisation, the organisation's farms the user has access to. Use `scope=own` to only retrieve the projects the user created. The list is [paged](#lists-and-paging).

### HTTP Request

`GET https://api.solvi.ag/v1/projects`

### Parameters

| Parameter        |          | Description                                                                                                                                                                       |
| ---------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| field_geom       | optional | Boundaries of the field in [GeoJSON format](https://geojson.org/geojson-spec.html#introduction) and EPSG:4326 coordinate system (lonlat): a Polygon or MultiPolygon geometry, a Feature with such a geometry, or a FeatureCollection of such Features; only projects whose extent overlaps the boundaries are returned |
| field_id         | optional | Only return projects on the field with this id                                                                                                                                    |
| farm_id          | optional | Only return projects on the farm with this id                                                                                                                                     |
| status           | optional | Only return projects with this status: `processed`, `not_processed` or `failed`                                                                                                   |
| type             | optional | Only return projects of this type: `overlapping` or `stitched`                                                                                                                    |
| bands            | optional | Only return projects with these bands, for example `rgb`                                                                                                                          |
| crop             | optional | Only return projects with this crop                                                                                                                                               |
| plant_counts     | optional | Only return projects with plant counts in this state: `requested`, `review`, `bad_request` or `published`                                                                         |
| search           | optional | Only return projects whose identifier, field name or farm name contains this text                                                                                                 |
| survey_date_from | optional | Only return projects surveyed on or after this date, formatted as YYYY-MM-DD                                                                                                      |
| survey_date_to   | optional | Only return projects surveyed on or before this date, formatted as YYYY-MM-DD                                                                                                     |

The list also accepts the parameters common to [all lists](#lists-and-paging), including `scope` and `user_id`.

## Project outputs

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  'https://api.solvi.ag/v1/projects/<project-id>
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
    "url": "https://app.solvi.ag/projects/9999",
    "thumbnail_url": "https://app.solvi.ag/projects/9999/thumbnail.png",
    "resources": {
      "thumbnail": "https://app.solvi.ag/projects/9999/thumbnail.png",
      "ortho": "https://solvi-projects.s3.eu-west-1.amazonaws.com/uploads/ed12e5f6-b6c9-4df8-9522-1dbc29be854b/results/ortho.tiff?...",
      "dem": "https://solvi-projects-dev.s3.eu-west-1.amazonaws.com/uploads/ed12e5f6-b6c9-4df8-9522-1dbc29be854b/results/dem.tiff?..."
    },
    "metadata": null
  },
```

This endpoint retrieves projects created by the user or shared with the user by others.

The included `resources` are URLs that can be used to fetch the project outputs. These resource URLs are temporary, with a lifetime of 15 minutes before they expire.

### HTTP Request

`GET https://api.solvi.ag/v1/projects/<project_id>`

### Parameters

| Parameter  |          | Description                              |
| ---------- | -------- | ---------------------------------------- |
| project_id | required | Project ID given when project is created |

## Project tiled imagery

> Example request:

```shell
curl -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  'https://api.solvi.ag/v1/projects/<project-id>/tiles/<type>
```

> Example response:

```json
{
  "url": "https://ts2.solvi.nu/.../results/ortho.tiff/rgb/rgb/tile/{z}/{x}/{y}.png?token=...",
  "type": "tileset/orthomosaic",
  "resolution": 0.15961538646165616,
  "bounds": {
    "value": [
      -115.407127627406, 33.13991737241373, -115.39728408213948, 33.144345906954
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

`GET https://api.solvi.ag/v1/projects/<project-id>/tiles/<type>`

### Parameters

| Parameter  |          | Description                                                                                                                                |
| ---------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| project_id | required | Project ID given when project is created                                                                                                   |
| type       | required | Type of tiles to fetch; currently one of `ortho` for RGB orthomosaic, `dem` for colored elevation model or `plant_counts` for plant counts |

## Webhooks

Webhooks allow setting up integrations that subscribe to certain events from Solvi. When one of those events is triggered, an HTTP POST payload is sent to the webhook's configured URL. The available events are project status changes (event type `status_changed`), plant counts published (event type `plant_counts_published`) and task status changes (event type `task_status_changed`).

> Example webhook request payload

```json
{
  "project_id": 8993,
  "event_type": "status_changed",
  "old_status": "not_processed",
  "new_status": "processed"
}
```

The webhook is configured when [creating a project](#create-project) by specifing the `status_webhook` parameter, which should contain the URL of the webhook.

### Webhook events

The type of event that occured is determined by the `event_type` parameter of the request body. These event types are sent:

- `status_changed`, indicates that the project's status has changed
- `plant_counts_published`, triggered when plant counts have been published for the project
- `task_status_changed`, indicates that the status of a [task](#tasks) for the project has changed

#### Status changes

The status change messages contains `old_status`, the status of the project before this change, as well as `new_status`, the project's updated status.

There are three different project statuses:

- `not_processed` - the project has been created but not yet processed, project outputs will not be available
- `processed` - the project has been processed and its outputs are available
- `failed` - the processing for this project failed, outputs are not available

#### Plant Counts Published

Notifies that plant counts data has been published for this project. The results can be fetched using the tiles endpoint with `plant_counts` as type.

#### Task status changes

> Example task status change payload

```json
{
  "project_id": 8993,
  "event_type": "task_status_changed",
  "task": {
    "id": "3f0c1d6e2a9b4c8d7e6f5a4b3c2d1e0f9a8b7c6d",
    "type": "zonal_statistics",
    "project_id": 8993,
    "status": "succeeded",
    "created_at": "2026-12-01T09:00:00.000Z",
    "updated_at": "2026-12-01T09:03:12.000Z",
    "result": {
      "url": "https://api.solvi.ag/v1/projects/8993/zonal_statistics"
    }
  },
  "old_status": "running",
  "new_status": "succeeded"
}
```

Sent when a [task](#tasks) for the project changes status, for example from `queued` to `running`, from `running` to `succeeded`, or from `succeeded` back to `queued` when a finished task [runs again](#running-the-same-task-again). The message contains `old_status` and `new_status`, the task's status before and after the change, and `task`, the task as returned by the [get task](#get-a-task) endpoint. No message is sent when a task is created.

A task that runs out of time is reported as `failed` by the get task endpoint as soon as its time is up, but the webhook for it can arrive later.

### Securing webhooks

To ensure that Solvi is the sender of the webhook requests, you can optionally also specify a secret token when registering the webhook, by using the `webhook_secret` parameter: the secret can be any string of your choosing.

> Example signature header

```
X-Solvi-Signature: sha1=494e5dbdd1afbe4d44091bf86872b5eb4b9133e5
```

When a secret token has been specified, Solvi will include the HTTP header `X-Solvi-Signature`, which will contain an HMAC-SHA1 signature of the body.

This follows the same pattern as [securing webhooks on GitHub](https://docs.github.com/en/developers/webhooks-and-events/securing-your-webhooks), except for using the header `X-Solvi-Signature` instead.

# Plant Counts

Plant counts are created either automatically with Plant AI in Solvi's web app, or done for you by Solvi personnel. The plant counts API orders a "done for you" plant count for a project, lists the project's plant counts and fetches their results.

Ordering a "done for you" plant count is a two step flow: first quote the plant count, which calculates its cost without charging anything, then accept the quote, which charges the cost and places the order. Solvi personnel then perform the count and publish the results, which can take up to a few business days; poll the status with the endpoints below, or use a [webhook](#webhooks) with the `plant_counts_published` event to get notified.

A project can have at most one plant count.

## Listing the Plant AI Crops

> Example request:

```shell
curl
  -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  https://api.solvi.ag/v1/plantai_crops
```

> Example response:

```json
{
  "crops": [
    {
      "id": "corn",
      "name": "Corn"
    },
    {
      "id": "sunflower",
      "name": "Sunflower"
    },
    {
      "id": "other",
      "name": "Other",
      "freeform": true
    }
  ]
}
```

Lists the crop choices a plant count can be requested for. A crop's `id` is the value the quote endpoint's `crop` parameter takes.

| Field    | Description                                                                                                                 |
| -------- | ----------------------------------------------------------------------------------------------------------------------------- |
| id       | The choice's id, passed as the quote's `crop` parameter                                                                     |
| name     | The crop's display name                                                                                                     |
| freeform | When `true`, choosing this crop requires naming the crop with the quote's `crop_name` parameter; absent on the other choices |

### HTTP Request

`GET https://api.solvi.ag/v1/plantai_crops`

## Quoting a Plant Count

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{
    "detection_type": "dfy",
    "mode": "basic",
    "crop": "corn"
  }'
  https://api.solvi.ag/v1/projects/<project-id>/plant_counts
```

> Example response:

```json
{
  "status": "quoted",
  "transaction_id": "eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiJ...",
  "expires_at": "2026-11-14T09:18:37.655Z",
  "detection_type": "dfy",
  "mode": "basic",
  "crop": {
    "id": "corn",
    "name": "Corn"
  },
  "area": 248300,
  "billed_hectares": 25,
  "cost": {
    "credit": {
      "amount": 125,
      "sufficient": true
    }
  }
}
```

Quotes a "done for you" plant count for a project: calculates the cost of the detection and returns a `transaction_id` for accepting the quote. Nothing is charged and no plant count is created until the quote is [accepted](#accepting-a-plant-count-quote).

The crop to count is chosen from [the crop listing](#listing-the-plant-ai-crops). A crop that is not on the list is ordered with the freeform choice `"crop": "other"`, naming the crop in the `crop_name` parameter; `crop_name` is only accepted together with `"other"`, and is trimmed and lowercased by the server. Weed detection has no crop: with the `weed` mode, `crop` and `crop_name` must be omitted. The response's `crop` echoes the choice, with the normalized `crop_name` as the `name` of a freeform crop; a weed quote has no `crop` field.

The detection area can be given with the optional `geojson` parameter. When it is omitted, the area defaults to the boundaries of the project's field or, if the field has no saved boundaries, the extent of the project's imagery. The response's `area` is the detection area in square meters, and the cost is calculated from `billed_hectares`, the area rounded up to whole hectares.

The quote is valid until `expires_at`, 24 hours after it was made; after that, a new quote has to be requested. Accepting the quote charges exactly the quoted cost, and the `sufficient` property tells whether the account's current balance covers it.

Quoting requires the project to be successfully processed and to not already have a plant count. Error responses:

- `400`: invalid parameters: a `detection_type` other than `dfy`, an unknown `mode` or `crop`, a missing or blank `crop_name` when `crop` is `other`, a `crop_name` together with a listed crop, a `crop` or `crop_name` with the `weed` mode, invalid `geojson` or coordinates outside WGS84, a project without a field, or an empty detection area
- `409`: the project has not been successfully processed, or already has a plant count

### HTTP Request

`POST https://api.solvi.ag/v1/projects/<project-id>/plant_counts`

### Parameters

| Parameter      |          | Description                                                                                                                                                                                                                              |
| -------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| detection_type | required | Type of plant count to order; only `dfy` ("done for you") can be ordered through the API                                                                                                                                                 |
| mode           | required | Detection mode: one of `plant-level-data`, `basic` or `weed`                                                                                                                                                                             |
| crop           | required | The crop to count: a crop id from [the crop listing](#listing-the-plant-ai-crops), or `other` for a crop that is not listed. Required, except with the `weed` mode, where it must be omitted                                              |
| crop_name      | optional | Name of the crop to count; required (and only accepted) when `crop` is `other`                                                                                                                                                           |
| geojson        | optional | Detection area as GeoJSON: a FeatureCollection, a Feature or a bare geometry, with Polygon or MultiPolygon geometries in WGS84 (EPSG:4326) coordinates. Defaults to the boundaries of the project's field, or the extent of its imagery |

## Accepting a Plant Count Quote

> Example request:

```shell
curl -X POST
  -H "Authorization: Bearer <user-jwt-token>"
  -H "Content-Type: application/json"
  -d '{
    "transaction_id": "eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiJ..."
  }'
  https://api.solvi.ag/v1/projects/<project-id>/plant_counts/accept
```

> Example response:

```json
{
  "status": "created",
  "id": "88a2af8601cf2b7c4e14979f8df73bca7352ccce"
}
```

Accepts a quote: charges the quoted cost from the project owner's credits and orders the plant count. Solvi personnel are notified and will perform the count. The returned `id` is the plant count's id, which works with the listing and data endpoints below: the plant count's status is `processing` until the results have been published.

A quote made with a `geojson` parameter must be accepted with the same `geojson` document; a quote made from the project's field boundaries or imagery extent is accepted without one, since those are read again when accepting. Either way, the detection area must still match the quoted area: if it doesn't, for example because the field's boundaries changed after the quote, the request is refused with a `400` response, and the plant count has to be quoted again.

A `transaction_id` charges at most once: accepting the same quote again returns a `409` response without charging. Error responses:

- `400`: a missing, invalid or expired `transaction_id`, a quote made for another project or user, a quoted crop that is no longer available, a project without a field, or a detection area that does not match the quote
- `402`: the account does not have enough credits to pay the quoted cost; the response body gives the required amount
- `409`: the quote has already been accepted, or the project already has a plant count

### HTTP Request

`POST https://api.solvi.ag/v1/projects/<project-id>/plant_counts/accept`

### Parameters

| Parameter      |          | Description                                                                                                       |
| -------------- | -------- | ----------------------------------------------------------------------------------------------------------------- |
| transaction_id | required | The `transaction_id` returned by the quote                                                                        |
| geojson        | optional | The same `geojson` document the quote was made with; required when the quote had one, and omitted when it had none |

## Listing a Project's Plant Counts

> Example request:

```shell
curl
  -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  https://api.solvi.ag/v1/projects/<project-id>/plant_counts
```

> Example response:

```json
[
  {
    "id": "88a2af8601cf2b7c4e14979f8df73bca7352ccce",
    "status": "completed",
    "created_at": "2019-02-08T09:18:37.655Z"
  },
  {
    "id": "34a2af8601cf2b7c4e14979f8df73bca7352aef43",
    "status": "published",
    "created_at": "2021-03-12T14:33:31.241Z"
  }
]
```

Lists all plant counts for a project with their id and status. Status is one of:

- `processing`: the detection has not yet completed
- `completed`: detection has finished and can be viewed
- `published`: this detection is the one currently published for the project
- `fail`: detection failed

A project with a "done for you" plant count lists a single plant count, which is `processing` until Solvi has published it.

### HTTP Request

`GET https://api.solvi.ag/v1/projects/<project-id>/plant_counts`

## Plant Count data

> Example request:

```shell
curl
  -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  https://api.solvi.ag/v1/projects/<project-id>/plant_counts/<plant-count-id>
```

> Example response:

```json
{
  "status": "completed",
  "results": [
    "https://solvi-projects.s3.eu-west-1.amazonaws.com/uploads/..."
  ]
}
```

Gets the status of a plant count, and the results of a completed one. The plant count id is the `id` given when listing a project's plant counts.

Depending on status of the plant count, different HTTP status codes will be used for the response:

- `200`: completed successfully and the body contains JSON information about the result
- `202`: processing is not yet complete
- `404`: unknown plant count id or the plant count's result has expired and been removed
- `500`: an internal error occurred during processing

When processing has completed, the response will contain a `results` property which lists URLs that can be used to get the detection results. These URLs are valid for at least 24 hours.

The results of a plant count that is not published are kept in storage for one week. After this time, the results are removed from storage and requesting the results will return a 404 code. The results of a published plant count are kept in storage.

Note that for "done for you" plant counts in legacy projects, processed before 2023-01-01, the returned result _may_
be a single ZIP archive file containing a Shapefile. In newer projects, the returned data will always be uncompressed data in multiple files.

### HTTP Request

`GET https://api.solvi.ag/v1/projects/<project-id>/plant_counts/<plant-count-id>`

# Tasks

Some operations take too long to answer in a single request. Instead, the request starts a _task_ that runs in the background, and responds with `202 Accepted` and the task, with a `Location` header holding the task's URL. The task is then followed until it finishes, either by fetching it with the [get task](#get-a-task) endpoint, or by receiving `task_status_changed` messages on the project's [webhook](#webhooks).

[Calculating zonal statistics](#calculating-statistics-for-zones) is currently the only operation that runs as a task.

## The task object

> Example task:

```json
{
  "id": "3f0c1d6e2a9b4c8d7e6f5a4b3c2d1e0f9a8b7c6d",
  "type": "zonal_statistics",
  "project_id": 8993,
  "status": "succeeded",
  "created_at": "2026-12-01T09:00:00.000Z",
  "updated_at": "2026-12-01T09:03:12.000Z",
  "result": {
    "url": "https://api.solvi.ag/v1/projects/8993/zonal_statistics"
  }
}
```

> Example failed task:

```json
{
  "id": "3f0c1d6e2a9b4c8d7e6f5a4b3c2d1e0f9a8b7c6d",
  "type": "zonal_statistics",
  "project_id": 8993,
  "status": "failed",
  "created_at": "2026-12-01T09:00:00.000Z",
  "updated_at": "2026-12-01T09:00:41.000Z",
  "error": {
    "code": "invalid_request",
    "message": "The statistics could not be calculated for these zones"
  }
}
```

| Property   | Description                                                                                                         |
| ---------- | ------------------------------------------------------------------------------------------------------------------- |
| id         | The task's id, a string                                                                                             |
| type       | The kind of work the task does; currently always `zonal_statistics`                                                 |
| project_id | The project the task works on                                                                                       |
| status     | One of `queued`, `running`, `succeeded` and `failed`, see below                                                      |
| created_at | When the task was created                                                                                           |
| updated_at | When the task last changed                                                                                          |
| result     | Only for a `succeeded` task: an object whose `url` is the endpoint to fetch the result from                          |
| error      | Only for a `failed` task: an object with a `code` and a `message` describing why the task failed                     |

A task's `status` is one of:

- `queued` - the task is waiting to start
- `running` - the task is running
- `succeeded` - the task finished, and its result can be fetched from `result.url` with the same authentication as other requests. The URL returns what is currently saved with the project, so a later request that changes the zones or recalculates the statistics changes what it returns; fetch the result before starting another request if you need it as it was.
- `failed` - the task did not finish; `error` says why

A failed task's `error.code` is one of:

- `invalid_request` - the task's input could not be used, for example zones the statistics can't be calculated for; `message` says what was wrong
- `conflict` - the project's data changed while the task was running, so its result was not saved; start the task again
- `timeout` - the task did not finish within two hours of being started
- `internal_error` - an error occurred in Solvi; please contact support if it persists

### Running the same task again

A task's id is derived from the project and the request that started it. Repeating a request while its task is `queued` or `running` does not start another task: the response is `202 Accepted` with the task already running. Repeating it after the task has finished runs the task again, under the same id.

### Limits

A user can have at most three tasks `queued` or `running` at the same time. A request that would start a fourth gets a `429 Too Many Requests` response with a `Retry-After` header, giving the number of seconds to wait before trying again.

## Get a task

> Example request:

```shell
curl
  -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  https://api.solvi.ag/v2/tasks/<task-id>
```

> Example response:

```json
{
  "id": "3f0c1d6e2a9b4c8d7e6f5a4b3c2d1e0f9a8b7c6d",
  "type": "zonal_statistics",
  "project_id": 8993,
  "status": "running",
  "created_at": "2026-12-01T09:00:00.000Z",
  "updated_at": "2026-12-01T09:00:02.000Z"
}
```

Fetches a [task](#the-task-object). Any user with access to the task's project can fetch it. A `404 Not found` is returned if the task does not exist, or the user doesn't have access to its project.

When polling a task until it finishes, wait a few seconds between requests.

### HTTP Request

`GET https://api.solvi.ag/v2/tasks/<task-id>`

## List tasks

> Example request:

```shell
curl
  -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  "https://api.solvi.ag/v2/tasks?project_id=8993&status=running"
```

> Example response:

```json
{
  "tasks": [
    {
      "id": "3f0c1d6e2a9b4c8d7e6f5a4b3c2d1e0f9a8b7c6d",
      "type": "zonal_statistics",
      "project_id": 8993,
      "status": "running",
      "created_at": "2026-12-01T09:00:00.000Z",
      "updated_at": "2026-12-01T09:00:02.000Z"
    }
  ],
  "continuation_token": null
}
```

Lists the tasks started by the user, oldest first. Tasks for projects the user no longer has access to are left out.

The response is an object with the tasks in `tasks`. Unlike the [lists of the v1 API](#lists-and-paging), the next page is given by `continuation_token` in the response body: when it is not `null`, more tasks may follow, and are fetched by repeating the request with the `continuation_token` parameter set to its value. A page can hold fewer tasks than `limit`, even none, while more follow; continue until `continuation_token` is `null`.

### HTTP Request

`GET https://api.solvi.ag/v2/tasks`

### Parameters

| Parameter          |          | Description                                                                                                       |
| ------------------ | -------- | ----------------------------------------------------------------------------------------------------------------- |
| project_id         | optional | Only return tasks for this project                                                                                |
| type               | optional | Only return tasks of this type, for example `zonal_statistics`                                                    |
| status             | optional | Only return tasks with this status: `queued`, `running`, `succeeded` or `failed`                                  |
| limit              | optional | The maximum number of tasks to return, 1-1000; default is 1000                                                    |
| continuation_token | optional | Continue the list after the previous page, with the `continuation_token` of the previous response                 |

An unknown `type` or `status`, or an invalid `limit` or `continuation_token`, gives a `400 Bad request` response.

# Zonal Statistics

The Zonal Statistics API lets you create zones in a project and calculate various statistics for them: primarily various vegetation indices as well as plant count statistics for projects where plant counts have been published.

Statistics are calculated in a [task](#tasks): the request that asks for them responds right away, and the statistics are saved with the project's zones when the task has finished.

## Calculating statistics for zones

> Example request:

```shell
curl -i -X POST
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
            "type": "Polygon",
            "coordinates": [
              [
                [-115.40068410376747, 33.14301501895726],
                [-115.40068410376747, 33.142561119287876],
                [-115.4001781509446, 33.142561119287876],
                [-115.4001781509446, 33.14301501895726],
                [-115.40068410376747, 33.14301501895726]
              ]
            ]
          }
        }
      ]
    }
  }'
  https://api.solvi.ag/v2/projects/<project-id>/zonal_statistics
```

> Example response:

```
HTTP/1.1 202 Accepted
Location: https://api.solvi.ag/v2/tasks/3f0c1d6e2a9b4c8d7e6f5a4b3c2d1e0f9a8b7c6d
```

```json
{
  "id": "3f0c1d6e2a9b4c8d7e6f5a4b3c2d1e0f9a8b7c6d",
  "type": "zonal_statistics",
  "project_id": 8993,
  "status": "queued",
  "created_at": "2026-12-01T09:00:00.000Z",
  "updated_at": "2026-12-01T09:00:00.000Z"
}
```

Sets the zones for a project, calculates statistics for them, or both.

The zones are given by the `geojson` parameter: a GeoJSON FeatureCollection with Polygon features, at most 10000 of them. Other GeoJSON objects or invalid GeoJSON result in a `400 Bad request` response. The zones replace the zones saved with the project. If `geojson` is the only parameter, no statistics are calculated: the zones are saved, and the response is `200 OK` with the zones as a GeoJSON FeatureCollection.

Statistics are asked for with the `indices` and `extra_data` parameters. The `indices` parameter lists the vegetation indices to calculate for the zones, as lowercase abbreviations: for example, the VARI index should be listed as `"vari"`. The available indices vary depending on the project's bands: for example, NDVI is only available if the project's orthophoto has red and near-infrared (NIR) bands. For projects with published plant counts, plant count statistics can be calculated by setting `extra_data` to `{"plant_counts": true}`.

When statistics are asked for, the request is checked and any posted zones are saved right away, and a [task](#tasks) is started to calculate the statistics. The response is `202 Accepted` with the task, and a `Location` header with the task's URL. When the task has succeeded, the zones with their statistics are saved with the project; they can be viewed on Solvi's web site, and fetched with the [GET endpoint](#getting-zonal-statistics-for-a-project), which is also the task's `result.url`.

If `geojson` is omitted, the statistics are calculated for the zones already saved with the project. A `400 Bad request` is returned if the project has no saved zones, and a `409 Conflict` if the saved zones were created with an old version of Solvi's web site; opening the project on the web site updates them.

Unknown indices, indices that can't be calculated for the project, and plant count statistics for a project without published plant counts result in a `400 Bad request` response, and nothing is saved. A request without any of the three parameters is also answered with `400 Bad request`.

If the project's zones are changed, for example on Solvi's web site, while the task is running, the task fails with the error code `conflict` and its statistics are not saved.

A `404 Not found` is returned if the project does not exist, or the user is not allowed to edit it.

### HTTP Request

`POST https://api.solvi.ag/v2/projects/<project-id>/zonal_statistics`

### Parameters

| Parameter  |          | Description                                                                                                                                                  |
| ---------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| geojson    | optional | GeoJSON FeatureCollection representing the zone features; defaults to the zones saved with the project                                                       |
| indices    | optional | List of vegetation indices to calculate; index names should be lower case strings                                                                            |
| extra_data | optional | Extra data layers to calculate statistics for, expressed as an object with string layer names as keys and boolean values to indicate if layer should be used |

At least one of the parameters must be given.

<a id="creating-and-calculating-statistics-for-zones"></a>

## Creating and calculating statistics for zones (deprecated)

<aside class="warning">This endpoint is deprecated and stops working on 1 June 2027. Use <a href="#calculating-statistics-for-zones">POST /v2/projects/&lt;project-id&gt;/zonal_statistics</a> instead, which calculates the statistics in a task rather than keeping the request open. Responses from this endpoint carry the <code>Deprecation</code> and <code>Sunset</code> headers (RFC 9745 and RFC 8594).</aside>

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
  https://api.solvi.ag/v1/projects/<project-id>/zonal_statistics
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

Sets the zones for a project and calculates the provided statistics, if one of the `indices` or `extra_data` parameters are included. The zones should be provided through the `geojson` parameter: it should contain a GeoJSON FeatureCollection with Polygon features. Other GeoJSON objects or invalid GeoJSON will result in a `400 Bad request` response.

If `geojson` is omitted, the statistics are calculated for the zones already saved with the project. A `400 Bad request` is returned if the project has no saved zones, and a `409 Conflict` if the saved zones were created with an old version of Solvi's web site; opening the project on the web site updates them.

In addition to the zone features, the `indices` parameter lists the vegetation indices that should be calculated for the zones. The indices should be listed as lowercase abbreviations, for example VARI index should be listed as `"vari"`. Note that the available indices vary depending on the project's bands: for example, NDVI is only available if the project's orthophoto has red and near-infrared (NIR) bands. Listing unknown indices or indices that can't be calculated will also result in a `400 Bad request` response. If `indices` are omitted, no indices will be calculated.

For projects with published plant counts, plant count statistics can also be calculated by the optional parameter `extra_data`, which in this case should be set to `{"plant_counts": true}`.

The endpoint responds with a GeoJSON FeatureCollection with the same geometries that were provided, but with properties added for the calculated statistics (if any). This response is also saved with the project, and can be viewed on Solvi's web site, and also later fetched with the GET endpoint below.

At most 10000 zones can be posted in one request. The request is answered when the statistics have been calculated, which for large projects or many zones can take a long time: if the calculation does not finish within 55 minutes, a `504 Gateway Timeout` is returned. A user can have at most two requests to this endpoint running at the same time; a third gets a `429 Too Many Requests` response with a `Retry-After` header, giving the number of seconds to wait before trying again.

### HTTP Request

`POST https://api.solvi.ag/v1/projects/<project-id>/zonal_statistics`

### Parameters

| Parameter  |          | Description                                                                                                                                                  |
| ---------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| geojson    | optional | GeoJSON FeatureCollection representing the zone features; defaults to the zones saved with the project                                                       |
| indices    | optional | List in vegetation indices to calculate; index names should be lower case strings                                                                            |
| extra_data | optional | Extra data layers to calculate statistics for, expressed as an object with string layer names as keys and boolean values to indicate if layer should be used |

## Getting Zonal Statistics for a project

> Example request:

```shell
curl
  -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  https://api.solvi.ag/v1/projects/<project-id>/zonal_statistics
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

Fetches any saved zonal statistics for a project. The zones are returned as a GeoJSON FeatureCollection with statistics in each feature's `properties`. If no zones are defined for this project, a `404 Not found` is returned. If the zones were created with an old version of Solvi's web site, a `409 Conflict` is returned; opening the project on the web site updates them, after which this endpoint returns them.

# Transactions

The Transactions API allows you to retrieve the user's credit and upload transaction history, including purchases and usage from project uploads and PlantAI detections.

## Get transactions

> Example request:

```shell
curl
  -X GET
  -H "Authorization: Bearer <user-jwt-token>"
  https://api.solvi.ag/v1/transactions
```

> Example response:

```json
[
  {
    "datetime": "2023-03-01T09:00:00.000Z",
    "unit": "credits",
    "amount": 1000,
    "type": "purchase"
  },
  {
    "datetime": "2023-03-14T14:20:00.000Z",
    "unit": "uploads",
    "amount": -1,
    "type": "upload",
    "project": {
      "project_id": 1233,
      "name": "Corn Field",
      "status": "processed",
      ...
    }
  },
  {
    "datetime": "2023-03-15T10:30:00.000Z",
    "unit": "credits",
    "amount": -25,
    "type": "plantai_detection",
    "area_ha": 2.5,
    "project": {
      "project_id": 1234,
      "name": "Wheat Field Survey",
      "status": "processed",
      ...
    }
  }
]
```

Retrieves the transaction history for the authenticated user, showing both usage (negative amounts) and purchases (positive amounts). By default, returns transactions from the last year. Transactions can be filtered by date range using the `from` and `to` parameters.

Usage transactions include details about the related project and, for PlantAI detections, the analyzed area in hectares.

### HTTP Request

`GET https://api.solvi.ag/v1/transactions`

### Parameters

| Parameter |          | Description                                                                                                                             |
| --------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| from      | optional | Start date to filter transactions (ISO 8601 format, e.g., "2023-01-01"); defaults to 1 year ago if neither `from` nor `to` is specified |
| to        | optional | End date to filter transactions (ISO 8601 format, e.g., "2023-12-31")                                                                   |

### Response

For usage related to a specific project, information about the project is included in the response. See [Get Projects](#get-projects) for full information about the details included.
