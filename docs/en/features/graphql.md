# API Documentation

* [Characteristics](#characteristics)
* [GraphQL](#graphql)
* [Making API requests](#making-api-requests)
  * [Supported clients](#supported-clients)
    * [GraphiQL](#graphiql)
    * [Postman](#postman)
    * [HTTP libraries](#http-libraries)
* [Available information](#available-information)
* [Examples of queries](#examples-of-queries)
  * [Request a single record from a collection](#request-a-single-record-from-a-collection)
  * [Request a complete collection](#request-a-complete-collection)
    * [Pagination](#pagination)
  * [Accessing several resources in a single request](#accessing-several-resources-in-a-single-request)
* [Security limitations](#security-limitations)
  * [Example of a query which is too deep](#example-of-a-query-which-is-too-deep)
  * [Example of a query which is too complex](#example-of-a-query-which-is-too-complex)
* [Code examples](#code-examples)
  * [Simple example](#simple-example)
  * [Example with pagination](#example-with-pagination)

## Characteristics

* Read-only API
* Public access, no authentication needed
* Uses GraphQL technology:
  * Maximum page size (and the default) is 25 records
  * Maximum query depth is set at 8 levels
  * A maximum of two collections can be requested within the same query
  * Support for GET requests (query must be inside the *query string*) and POST requests (query must be within the *body*, encoded as `application/json` or `application/graphql`)

## GraphQL

The Consul Democracy API uses [GraphQL](http://graphql.org), specifically the [Ruby implementation](http://graphql-ruby.org/). If you're not familiar with this kind of APIs, we recommended you to check the [GraphQL official documentation](https://graphql.org/learn/).

One of the characteristics that differentiates a REST API from a GraphQL one is that with the latter one it's possible for the client to build its own *custom queries*, so the server will only return information in which we're interested.

GraphQL queries are written following a format which resembles JSON. For example:

```graphql
{
  proposal(id: 1) {
    id,
    title,
    public_author {
      id,
      username
    }
  }
}
```

Responses are formatted in JSON:

```json
{
  "data": {
    "proposal": {
      "id": 1,
      "title": "Increase the amount of green spaces",
      "public_author": {
        "id": 2,
        "username": "abetterworld"
      }
    }
  }
}
```

## Making API requests

Following [the official recommendations](http://graphql.org/learn/serving-over-http/), the Consul Democracy API supports the following kind of requests:

* GET requests, with the query inside the *query string*.
* POST requests
  * With the query inside the *body*, with `Content-Type: application/json`
  * With the query inside the *body*, with `Content-Type: application/graphql`

### Supported clients

Since this is an API that works through HTTP, any tool capable of making this kind of requests is capable of querying the API.

This section presents a few examples about how to make requests using:

* GraphiQL
* Chrome extensions like Postman
* Any HTTP library

#### GraphiQL

[GraphiQL](https://github.com/graphql/graphiql) is a browser interface for making queries against a GraphQL API. It's also an additional source of documentation. Consul Democracy uses the [graphiql-rails](https://github.com/rmosolgo/graphiql-rails) to access this interface at `/graphiql`; it's the best way to get familiar with GraphQL-based APIs.

![The interface of GraphiQL](../../img/graphql/graphiql.png)

It's got three main panels:

* The left panel is used to write the query.
* The central panel shows the result of the request.
* The right panel (hideable) shows a documentation autogenerated from the models and fields exposed in the API.

#### Postman

Here's an example of a `GET` request, with the query as part of the *query string*:

![GET request with Postman, showing the query on the browser's URL](../../img/graphql/graphql-postman-get.png)

And here's an example of a `POST` request, with the query as part of the *body* and encoded as `application/json`:

!["Headers" tabe in Postman, with "Content-Type" set to "application/json"](../../img/graphql/graphql-postman-post-headers.png)

The query must be located inside a valid JSON document, as the value of the `"query"` key:

!["Body" tab in Postman, with the query inside inside the "query" key](../../img/graphql/graphql-postman-post-body.png)

#### HTTP libraries

You can use any of the HTTP libraries available for most programming languages.

**IMPORTANT**: Some servers might use security protocols that will make it necessary to include a *User Agent* header from a web browser so the request is not rejected. For example:

`User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/116.0`

## Available information

The `app/graphql/types/` folder contains a complete list of all the models (and their attributes) which are currently being exposed in the API.

The models are the following:

| Model                   | Description                   |
| ----------------------- | ----------------------------- |
| `User`                  | Users                         |
| `Debate`                | Debates                       |
| `Proposal`              | Proposals                     |
| `Budget`                | Participatory budgets         |
| `Budget::Investment`    | Budget investments            |
| `Comment`               | Comments on debates, proposals and other comments |
| `Milestone`             | Proposals, investments and processes milestones |
| `Geozone`               | Geozones (districts)          |
| `ProposalNotification`  | Notifications related to proposals |
| `Tag`                   | Tags on debates and proposals |
| `Vote`                  | Information related to votes  |

## Examples of queries

### Request a single record from a collection

```graphql
{
  proposal(id: 2) {
    id,
    title,
    comments_count
  }
}
```

Response:

```json
{
  "data": {
    "proposal": {
      "id": 2,
      "title": "Create a dog-friendly area near the beach",
      "comments_count": 10
    }
  }
}
```

### Request a complete collection

```graphql
{
  proposals {
    edges {
      node {
        title
      }
    }
  }
}
```

Response:

```json
{
  "data": {
    "proposals": {
      "edges": [
        {
          "node": {
            "title": "Bus stop near the school"
          }
        },
        {
          "node": {
            "title": "Improve the lighting in the football stadium"
          }
        }
      ]
    }
  }
}
```

#### Pagination

The maximum (and default) number of records that each page contains is set to 25. In order to navigate through the different pages, it's necessary to request `endCursor` information:

```graphql
{
  proposals(first: 25) {
    pageInfo {
      hasNextPage
      endCursor
    }
    edges {
      node {
        title
      }
    }
  }
}

```

The response:

```json
{
  "data": {
    "proposals": {
      "pageInfo": {
        "hasNextPage": true,
        "endCursor": "NQ=="
      },
      "edges": [
        # ...
      ]
    }
  }
}
```

To retrieve the next page, pass the cursor received in the previous request as a parameter:

```graphql
{
  proposals(first: 25, after: "NQ==") {
    pageInfo {
      hasNextPage
      endCursor
    }
    edges {
      node {
        title
      }
    }
  }
}
```

### Accessing several resources in a single request

This query requests information about several models in a single request: `Proposal`, `User`, `Geozone` and `Comment`:

```graphql
{
  proposal(id: 15262) {
    id,
    title,
    public_author {
      username
    },
    geozone {
      name
    },
    comments(first: 2) {
      edges {
        node {
          body
        }
      }
    }
  }
}
```

## Security limitations

Allowing a client to customize queries is a major risk factor. If queries that are too complex were allowed, it would be possible to perform a DoS attack against the server.

There are three main mechanisms to prevent such abuses:

* Pagination of results
* Limit the maximum depth of the queries
* Limit the amount of information that is possible to request in a query

### Example of a query which is too deep

The maximum depth of queries is currently set at 8. Deeper queries (such as the following) will be rejected:

```graphql
{
  user(id: 1) {
    public_proposals {
      edges {
        node {
          id,
          title,
          comments {
            edges {
              node {
                body,
                public_author {
                  username
                }
              }
            }
          }
        }
      }
    }
  }
}
```

The response will look something like this:

```json
{
  "errors": [
    {
      "message": "Query has depth of 9, which exceeds max depth of 8"
    }
  ]
}
```

### Example of a query which is too complex

The main risk factor is the option to request multiple collections of resources in the same query. The maximum number of collections that can appear in the same query is limited to 2. The following query requests information from the `users`, `debates` and `proposals` collections, so it will be rejected:

```graphql
{
  users {
    edges {
      node {
        public_debates {
          edges {
            node {
              title
            }
          }
        },
        public_proposals {
          edges {
            node {
              title
            }
          }
        }
      }
    }
  }
}
```

The response will look something like this:

```json
{
  "errors": [
    {
      "message": "Query has complexity of 3008, which exceeds max complexity of 2500"
    }
  ]
}
```

However, it is possible to request information belonging to more than two models in a single query, as long as you do not try to access the entire collection. For example, the following query that accesses the `User`, `Proposal` and `Geozone` models is valid:

```graphql
{
  user(id: 468501) {
    id
    public_proposals {
      edges {
        node {
          title
          geozone {
            name
          }
        }
      }
    }
  }
}
```

The response:

```json
{
  "data": {
    "user": {
      "id": 468501,
      "public_proposals": {
        "edges": [
          {
            "node": {
              "title": "Make a discount to locals in sports centers",
              "geozone": {
                "name": "South area"
              }
            }
          }
        ]
      }
    }
  }
}
```

## Code examples

### Simple example

Here's a simple example of code accessing the Consul Democracy demo API:

```ruby
require "net/http"

API_ENDPOINT = "https://demo.consuldemocracy.org/graphql".freeze

def make_request(query_string)
  uri = URI(API_ENDPOINT)
  uri.query = URI.encode_www_form(query: query_string)
  request = Net::HTTP::Get.new(uri)
  request[:accept] = "application/json"

  Net::HTTP.start(uri.hostname, uri.port, use_ssl: true) do |https|
    https.request(request)
  end
end

query = <<-GRAPHQL
  {
    proposal(id: 1) {
        id,
        title,
        public_created_at
    }
  }
GRAPHQL

response = make_request(query)

puts "Response code: #{response.code}"
puts "Response body: #{response.body}"
```

### Example with pagination

And here is a more complex example using pagination, once again accessing the Consul Democracy demo API:

```ruby
require "net/http"
require "json"

API_ENDPOINT = "https://demo.consuldemocracy.org/graphql".freeze

def make_request(query_string)
  uri = URI(API_ENDPOINT)
  uri.query = URI.encode_www_form(query: query_string)
  request = Net::HTTP::Get.new(uri)
  request[:accept] = "application/json"

  Net::HTTP.start(uri.hostname, uri.port, use_ssl: true) do |https|
    https.request(request)
  end
end

def build_query(options = {})
  page_size = options[:page_size] || 25
  page_size_parameter = "first: #{page_size}"

  page_number = options[:page_number] || 0
  after_parameter = page_number.positive? ? ", after: \"#{options[:next_cursor]}\"" : ""

  <<-GRAPHQL
  {
    proposals(#{page_size_parameter}#{after_parameter}) {
      pageInfo {
        endCursor,
        hasNextPage
      },
      edges {
        node {
          id,
          title,
          public_created_at
        }
      }
    }
  }
  GRAPHQL
end

page_number = 0
next_cursor = nil
proposals   = []

loop do
  puts "> Requesting page #{page_number}"

  query = build_query(page_size: 25, page_number: page_number, next_cursor: next_cursor)
  response = make_request(query)

  response_hash  = JSON.parse(response.body)
  page_info      = response_hash["data"]["proposals"]["pageInfo"]
  has_next_page  = page_info["hasNextPage"]
  next_cursor    = page_info["endCursor"]
  proposal_edges = response_hash["data"]["proposals"]["edges"]

  puts "\tHTTP code: #{response.code}"

  proposal_edges.each do |edge|
    proposals << edge["node"]
  end

  page_number += 1

  break unless has_next_page
end
```