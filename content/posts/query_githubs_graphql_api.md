+++
title = "Query Rust Repository Info on GitHub"
date = 2023-12-29T17:55:52-05:00
draft = true
+++

As one of the maintaners of [rustfmt] I've often wondered "how is rustfmt being used by everyday users?"
More specifically, I'm curious about what configuration options are being used.
As things currently stand, I don't have a lot of data points to help me answer that question.
Besides bug reports, the rustfmt team doesn't get much feedback from users on the options that they're using.
Among other things, I think it would be cool to know which non default options are the most popular, and whether or not nightly rustfmt exceeds stable rustfmt usage.

[rustfmt]: https://github.com/rust-lang/rustfmt

As I thought about this more and more I had a bit of a eurika moment. rustfmt is configured via a `rustfmt.toml` file, and most projects that
use rustfmt probably stick a `rustfmt.toml` somewhere in their project, often at the root.
For example, [rust-analyzer], [clippy], and [the Rust compiler] all have a `rustfmt.toml` in the root of their repositories.
What else do these projects have in common?
Besies all being affiliated with the [rust-lang organization], they're all hosted on GitHub!
In fact, [there are thousands of Rust projects hosted on Github], and I'm sure there are plenty that use rustfmt.

[rust-analyzer]: https://github.com/rust-lang/rust-analyzer
[clippy]: https://github.com/rust-lang/rust-clippy
[the Rust compiler]: https://github.com/rust-lang/rust
[rust-lang organization]: https://github.com/rust-lang
[there are thousands of Rust projects hosted on Github]: https://github.com/search?q=language%3ARust+&type=repositories

I think I found my dataset! Before getting too far along I want to say up front that I'm not looking for exact numbers on rustfmt usage.
Although there are a lot of rust projects on GitHub I know that's not a complete set of all the rust code that's out there.
That being said, even grabbing a fraction of the `rustfmt.toml` files on GitHub would be a huge leap forward in my opinion.
Overall, I think this will be a fun project to work on, and there might even be some practicle benefits that come from it too[^1].
Join me as I leverage GitHub and my software development skills to unravel the mystery of rustfmt usage.
I hope you'll find this posts intersting, entertaining, and maybe even a little educational.

[^1]: [rustfmt's stabalization process] tries to make sure that unstable options are "well tested, both in unit tests and, optimally, in real usage"
before they're promoted from unstable to stable. If the team had better data on the usage of unstable configurations that might help when
deciding if an option is ready for stabalization.

[rustfmt's stabalization process]: https://github.com/rust-lang/rustfmt/blob/master/Processes.md#stabilising-an-option

## Structure of project

As of right now, my idea is to build a CLI tool to help me collect, manage, and store details about rust projects and their `rustfmt.toml` files in a database.
I'll then analyze the data I collect to get a clearer picture of how rustfmt is configured accross a wide range of real projects.
Once I get a feel for what statistics might be useful and interesting I think I'll try and add a very simple user interface so that it's easier to explore the stats.

Although I have a rought idea of what I'd like to build, I'm keeping an open mind and I'd like this project to develop organically.
If I have new ideas along the way I'll try to incorporate them, and if I find that something isn't worth building (I'm looking at you UI), then I'll scrap it.
Here's the rough set of details I think I'll need to work out:

1. Query GitHub for Rust projects
   - What query will I have to make to get the data that I need?
   - I'm sure I'll need to hande pagination and rate limiting.
   - Deserializing the GraphQL response so I can work with it programatically.
2. Clone Rust Repos
   - Given that I'm building this project in Rust, I'd like a Rust interface for cloning the repositories.
     I haven't worked with the [git2] crate before, but it looks promising.
3. Search for `rustfmt.toml` files
   - This comes down to searching for the configuration on the file sytstem.
     The [rust_search] crate looks pretty cool, and from what I can tell, it has an easy to use API.
4. Extracting rustfmt.toml data
   - Once I've found the `rustfmt.toml` files in each repository it shouldn't be hard to grab the user configurations.
     I don't think I want to store the raw toml strings into the database, which leads me to my next point.
5. Store the data in the database
   - I'm leaninig towards using a relational database for this project. Most likely postgres.
     The metatdata for each repository is structured, while the use cofigurations are semi-strucuted.
     Because of that, I think storing each `rustfmt.toml` as [JSONB] would work well, and I can also add an [index on each configurations].
6. Analyze configuration usage
   - This phase will be very exploratory. Once I've actually collected the data I think it'll be easier to start posing questions and crafting queries.
7. Build a UI to explore statistics
   - I think a simple UI would be great to have, but it's the portion of the project that I'm least excited about right now so we'll see what happens.


[git2]: https://crates.io/crates/git2
[rust_search]: https://crates.io/crates/rust_search
[JSONB]: https://www.postgresql.org/docs/current/datatype-json.html
[index on each configurations]: https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING

The remainder of this post will focus on querying GitHub. I plan to touch on most of the points I've highlighted above in follow up articles.

## Query GitHub

The [GitHub GraphQL documentation] and the [Forming calls with GraphQL] section give a good overview of working with GitHub's GraphQL API.
I'll cover some basics just to make sure that we're all on the same page, but I may leave out some details.
Check those resources out if you'd like to learn more. If you're not familiar with GraphQL check out this [Introduction to GraphQL].


[GitHub GraphQL documentation]: https://docs.github.com/en/graphql
[Forming calls with GraphQL]: https://docs.github.com/en/graphql/guides/forming-calls-with-graphql
[Introduction to GraphQL]: https://graphql.org/learn/

In order to query GitHub's GraphQL API you'll need to [generate a personal access token] to use in the `Authorizaton` header.
Once you've got your token you should be able to run this simple CURL request, shamelessly taken from the GitHub docs, to illustrate how easy it is to call the GraphQL API.

[generate a personal access token]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic

```console
curl https://api.github.com/graphql \
-X POST \
-H "Authorization: Bearer {YOUR_GITHUB_API_TOKEN}" \
-d '{ "query": "query { viewer { login }}" }'
```

You should get a response back that looks something like this:

```json
{
  "data": {
    "viewer": {
        "login": "{YOUR_GITHUB_USERNAME}"
    }
  }
}
```

Not so difficult, right?
The API call boils down to an authenticated HTTP POST request with a JSON body containing the query we'd like the server to execute!
A few things to quickly note: First, the server responded with JSON data and second the structure of the response matches the query that was made.

### GraphQL Explorer

Another nice thing about GraphQL is that the server provides a strongly typed schema that we can use to explore what we're allowed to query.
I've worked with GraphQL in the past, but before I started on this project I didn't know what kind of data I'd be able to pull from GitHub.
I spent a decent amount of time using [GitHub's GraphQL Explorer] to craft my initial query for this project.
If you're looking to build anything with the GraphQL API I'd highly encourage you to test out differnt queries in the explorer, but if you just want to learn more about
certain types in the API then check out the [Object reference].
After some tinkering in the explorer this is the query I came up with:


[GitHub's GraphQL Explorer]: https://docs.github.com/en/graphql/overview/explorer
[Object reference]:https://docs.github.com/en/graphql/reference/objects

```graphql
query GitHubRepositorySearch(
  # The search string to look for. GitHub search syntax is supported.
  $gitHubSearchString: String!
  # Returns the first n elements from the list.
  $limit: Int!
  # Returns the elements in the list that come after the specified cursor.
  # Check the `endCursor` on the returned pageInfo
  $cursorOffset: String
  # Ordering options for language connections.
  $languageOrderBy: LanguageOrder!
) {
  search(first: $limit, after: $cursorOffset, query: $gitHubSearchString, type: REPOSITORY) {
    repositoryCount
    pageInfo {
      hasNextPage
      endCursor
    }
    nodes {
      ... on Repository {
        id
        nameWithOwner
        description
        url
        archivedAt
        isFork
        isLocked
        pushedAt
        updatedAt
        languages(first: 5, orderBy: $languageOrderBy) {
          totalCount
          totalSize
          edges {
            size
            node {
              name
            }
          }
        }
        defaultBranchRef {
          target {
            oid
          }
        }
      }
    }
  }
}
```

This query is a little more complicated than the one I showed earlier, but don't be intimidated, it's really not that bad.
I gave this query an optional name of `GitHubRepositorySearch`.
You'll notice that the query now has variables for `$gitHubSearchString`, `$limit`, `$offset`, and `$languageOrderBy`
which respectively have the types [`String!`], [`Int!`], [`String`], and [`LanguageOrder!`].
In GraphQL all values are optional by default, meaning that from a Rust user's perspective, `String` is really `Option<String>`.
To specify that a value is guaranteed to not be null you add an exclamation point as the end of the type name like `String!`.

[`String`]: https://docs.github.com/en/graphql/reference/scalars#string
[`String!`]: https://docs.github.com/en/graphql/reference/scalars#string
[`Int!`]:https://docs.github.com/en/graphql/reference/scalars#int
[`LanguageOrder!`]: https://docs.github.com/en/graphql/reference/input-objects#languageorder

The variables make it much easier to send dynamic queries to GitHub.
For example, I can change the `$gitHubSearchString` to have fewer or more filters, and I can control the number of repositories returned on each request by modifying the `$limit`.

Now that the query takes variables those need to be sent along in the POST request.
In addition to the `query` field, the JSON body now contains fields for the `variables` and `operationName`.
Here's an example of what the POST request's body might look like:

```json
{
  "query": "{The entire query from above^^^}",
  "operationName": "GitHubRepositorySearch",
  "variables": {
    "gitHubSearchString": "language:rust topic:rust stars:>=50 template:false archived:false",
    "limit": 10,
    "languageOrderBy": {
        "field": "SIZE",
        "direction": "DESC"
    }
  }
}
```

## Fetch GraphQL Data In Rust

Okay, time for some Rust code! Now that I know what query I want to send I want to write a small proof of concept (POC) program to fetch repository data from GitHub.
I'll omit the GraphQL query from the code as it's listed above. Just know that I'm storing it as a `const &str` named `GITHUB_REPOSITORY_QUERY`.


Here's what querying the GitHub API looks like.
I'm using the [reqwest] crate to send my GraphQL query to GitHub, [serde_json] to seriazlie the JSON paylaod, and [anyhow] to simplify error handling.

[reqwest]: https://crates.io/crates/reqwest
[serde_json]: https://crates.io/crates/serde_json
[anyhow]: https://crates.io/crates/anyhow

```rust
use reqwest::header;

const GITHUB_GRAPHQL_URL: &str = "https://api.github.com/graphql";

pub fn search_github_repositories(api_key: &str, user_agent: &str) -> anyhow::Result<String>> {
    let mut headers = header::HeaderMap::new();
    headers.insert(
        header::AUTHORIZATION,
        header::HeaderValue::from_str(&format!("Bearer {api_key}"))?,
    );

    let client = reqwest::blocking::ClientBuilder::new()
        .user_agent(user_agent)
        .default_headers(headers)
        .build()?;

    let body = serde_json::json!({
        "query": GITHUB_REPOSITORY_QUERY,
        "operationName": "GitHubRepositorySearch",
        "variables": {
            "gitHubSearchString": "language:rust topic:rust stars:>=50 template:false archived:false",
            "limit": 10,
            // TODO: add support for pagination
            "cursorOffset": None,
            "languageOrderBy": {"field": "SIZE", "direction": "DESC"}
        }
    });

    let resp = client
        .post(GITHUB_GRAPHQL_URL)
        .body(body.to_string())
        .send()?;

    Ok(resp.text()?)
}
```

The `main` function is pretty simple, and just sets things up to call `search_github_repositories`.
Note, I'm using [dotenv] to make it a little easier to set environment variables during development, like my `GITHUB_API_TOKEN`.

[dotenv]: https://crates.io/crates/dotenv

```rust
use anyhow::Context;

fn main() -> anyhow::Result<()> {
    dotenv::dotenv().ok();

    let github_api_token = std::env::var("GITHUB_API_TOKEN")
        .context("Must set GITHUB_API_TOKEN environment variable")?;

    let user_agent = std::env::var("GITHUB_USER_AGENT")
        .context("Must set GITHUB_USER_AGENT environment variable")?;

    let graphql_response = search_github_repositories(&github_api_token, &user_agent)?;
    println!("{graphql_response}");
}
```

Finally, what this has all been leading up to, the data! Here's the output from `cargo run`:

```json
{
   "data":{
      "search":{
         "repositoryCount":4327,
         "pageInfo":{
            "hasNextPage":true,
            "endCursor":"Y3Vyc29yOjEw"
         },
         "nodes":[
            {
               "id":"MDEwOlJlcG9zaXRvcnkxMzM0NDIzODQ=",
               "nameWithOwner":"denoland/deno",
               "description":"A modern runtime for JavaScript and TypeScript.",
               "url":"https://github.com/denoland/deno",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-15T16:41:05Z",
               "updatedAt":"2024-01-15T17:00:02Z",
               "languages":{
                  "totalCount":7,
                  "totalSize":12135341,
                  "edges":[
                     {
                        "size":5636933,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":3805663,
                        "node":{
                           "name":"JavaScript"
                        }
                     },
                     {
                        "size":2682078,
                        "node":{
                           "name":"TypeScript"
                        }
                     },
                     {
                        "size":5754,
                        "node":{
                           "name":"CSS"
                        }
                     },
                     {
                        "size":3902,
                        "node":{
                           "name":"C"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"72ecfe04198c5e912826663033a8963fbdea4521"
                  }
               }
            },
            {
               "id":"MDEwOlJlcG9zaXRvcnk3MjQ3MTI=",
               "nameWithOwner":"rust-lang/rust",
               "description":"Empowering everyone to build reliable and efficient software.",
               "url":"https://github.com/rust-lang/rust",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-15T17:00:31Z",
               "updatedAt":"2024-01-15T16:31:51Z",
               "languages":{
                  "totalCount":22,
                  "totalSize":82606569,
                  "edges":[
                     {
                        "size":79874988,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":457423,
                        "node":{
                           "name":"JavaScript"
                        }
                     },
                     {
                        "size":450748,
                        "node":{
                           "name":"Shell"
                        }
                     },
                     {
                        "size":327667,
                        "node":{
                           "name":"Fluent"
                        }
                     },
                     {
                        "size":276941,
                        "node":{
                           "name":"Python"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"1ead4761e9e2f056385768614c23ffa7acb6a19e"
                  }
               }
            },
            {
               "id":"MDEwOlJlcG9zaXRvcnkxOTY3MDE2MTk=",
               "nameWithOwner":"tauri-apps/tauri",
               "description":"Build smaller, faster, and more secure desktop applications with a web frontend.",
               "url":"https://github.com/tauri-apps/tauri",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-15T16:27:00Z",
               "updatedAt":"2024-01-15T17:05:17Z",
               "languages":{
                  "totalCount":19,
                  "totalSize":2092277,
                  "edges":[
                     {
                        "size":1670790,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":145469,
                        "node":{
                           "name":"TypeScript"
                        }
                     },
                     {
                        "size":81327,
                        "node":{
                           "name":"NSIS"
                        }
                     },
                     {
                        "size":62827,
                        "node":{
                           "name":"Kotlin"
                        }
                     },
                     {
                        "size":51094,
                        "node":{
                           "name":"JavaScript"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"5cb196976e5c84f2a34a34018e957424105cf42f"
                  }
               }
            },
            {
               "id":"MDEwOlJlcG9zaXRvcnkyOTkzNTQyMDc=",
               "nameWithOwner":"rustdesk/rustdesk",
               "description":"An open-source remote desktop, and alternative to TeamViewer.",
               "url":"https://github.com/rustdesk/rustdesk",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-15T07:59:34Z",
               "updatedAt":"2024-01-15T16:40:57Z",
               "languages":{
                  "totalCount":19,
                  "totalSize":5468689,
                  "edges":[
                     {
                        "size":3619870,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":1389949,
                        "node":{
                           "name":"Dart"
                        }
                     },
                     {
                        "size":120378,
                        "node":{
                           "name":"C"
                        }
                     },
                     {
                        "size":67214,
                        "node":{
                           "name":"Kotlin"
                        }
                     },
                     {
                        "size":49086,
                        "node":{
                           "name":"C++"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"58fd8780e537239fa07dd877ac48b19e3698d375"
                  }
               }
            },
            {
               "id":"MDEwOlJlcG9zaXRvcnk1MTk4MDQ1NQ==",
               "nameWithOwner":"alacritty/alacritty",
               "description":"A cross-platform, OpenGL terminal emulator.",
               "url":"https://github.com/alacritty/alacritty",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-14T16:25:16Z",
               "updatedAt":"2024-01-15T16:34:49Z",
               "languages":{
                  "totalCount":4,
                  "totalSize":1118149,
                  "edges":[
                     {
                        "size":1077462,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":26322,
                        "node":{
                           "name":"Shell"
                        }
                     },
                     {
                        "size":11108,
                        "node":{
                           "name":"GLSL"
                        }
                     },
                     {
                        "size":3257,
                        "node":{
                           "name":"Makefile"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"94ede16ee4af8869fd6415b3530c7e12c8681578"
                  }
               }
            },
            {
               "id":"MDEwOlJlcG9zaXRvcnk0MjQ4OTgyOQ==",
               "nameWithOwner":"rust-lang/rustlings",
               "description":":crab: Small exercises to get you used to reading and writing Rust code!",
               "url":"https://github.com/rust-lang/rustlings",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-15T14:52:37Z",
               "updatedAt":"2024-01-15T17:02:57Z",
               "languages":{
                  "totalCount":4,
                  "totalSize":163889,
                  "edges":[
                     {
                        "size":153370,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":4858,
                        "node":{
                           "name":"Shell"
                        }
                     },
                     {
                        "size":3045,
                        "node":{
                           "name":"PowerShell"
                        }
                     },
                     {
                        "size":2616,
                        "node":{
                           "name":"Nix"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"8d0aa11a35f822d044e6be89dc42840e0aea8858"
                  }
               }
            },
            {
               "id":"R_kgDOIksATQ",
               "nameWithOwner":"lencx/ChatGPT",
               "description":"🔮 ChatGPT Desktop Application (Mac, Windows and Linux)",
               "url":"https://github.com/lencx/ChatGPT",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-06T22:15:36Z",
               "updatedAt":"2024-01-15T15:25:55Z",
               "languages":{
                  "totalCount":5,
                  "totalSize":69191,
                  "edges":[
                     {
                        "size":65239,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":1502,
                        "node":{
                           "name":"HTML"
                        }
                     },
                     {
                        "size":1438,
                        "node":{
                           "name":"TypeScript"
                        }
                     },
                     {
                        "size":919,
                        "node":{
                           "name":"Ruby"
                        }
                     },
                     {
                        "size":93,
                        "node":{
                           "name":"Shell"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"fac5a4399ed553424be5388fe5eb24d5e5c0e98c"
                  }
               }
            },
            {
               "id":"MDEwOlJlcG9zaXRvcnkxMzA0NjQ5NjE=",
               "nameWithOwner":"sharkdp/bat",
               "description":"A cat(1) clone with wings.",
               "url":"https://github.com/sharkdp/bat",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-10T12:50:09Z",
               "updatedAt":"2024-01-15T16:13:06Z",
               "languages":{
                  "totalCount":4,
                  "totalSize":363923,
                  "edges":[
                     {
                        "size":351266,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":8968,
                        "node":{
                           "name":"Python"
                        }
                     },
                     {
                        "size":3651,
                        "node":{
                           "name":"Shell"
                        }
                     },
                     {
                        "size":38,
                        "node":{
                           "name":"Batchfile"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"cd81c7fa6bf0d061f455f67aae72dc5537f7851d"
                  }
               }
            },
            {
               "id":"MDEwOlJlcG9zaXRvcnk1MzYzMTk0NQ==",
               "nameWithOwner":"BurntSushi/ripgrep",
               "description":"ripgrep recursively searches directories for a regex pattern while respecting your gitignore",
               "url":"https://github.com/BurntSushi/ripgrep",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-14T14:32:31Z",
               "updatedAt":"2024-01-15T16:45:38Z",
               "languages":{
                  "totalCount":5,
                  "totalSize":1792536,
                  "edges":[
                     {
                        "size":1691876,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":47804,
                        "node":{
                           "name":"Python"
                        }
                     },
                     {
                        "size":38203,
                        "node":{
                           "name":"Shell"
                        }
                     },
                     {
                        "size":13849,
                        "node":{
                           "name":"Roff"
                        }
                     },
                     {
                        "size":804,
                        "node":{
                           "name":"Ruby"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"648a65f1976cc3b7eb66425024649d71d5befe1e"
                  }
               }
            },
            {
               "id":"MDEwOlJlcG9zaXRvcnkxMzA2ODgwMTE=",
               "nameWithOwner":"meilisearch/meilisearch",
               "description":"A lightning-fast search API that fits effortlessly into your apps, websites, and workflow",
               "url":"https://github.com/meilisearch/meilisearch",
               "archivedAt":null,
               "isFork":false,
               "isLocked":false,
               "pushedAt":"2024-01-15T16:52:06Z",
               "updatedAt":"2024-01-15T15:20:18Z",
               "languages":{
                  "totalCount":3,
                  "totalSize":3912274,
                  "edges":[
                     {
                        "size":3905234,
                        "node":{
                           "name":"Rust"
                        }
                     },
                     {
                        "size":5572,
                        "node":{
                           "name":"Shell"
                        }
                     },
                     {
                        "size":1468,
                        "node":{
                           "name":"Dockerfile"
                        }
                     }
                  ]
               },
               "defaultBranchRef":{
                  "target":{
                     "oid":"5204c0b60b384cbc79621b6b2176fca086069e8e"
                  }
               }
            }
         ]
      }
   }
}
```

## What's Next?

Query Rust repository details from GitHub, check ✅!
The first improvement I'd like to make is adding a nicer, type-safe API for the GraphQL response since all I'm doing now is printing out the JSON data.
It'll also be nice to add support for pagination, though that's not strictly necessary to move forward with the project.
There's still a lot to do, but I think this was a good first step.
[All the code for the project can be found here](https://github.com/ytmimi/Rustfmt-User-Config-Database)

If you've found any issues or typos with this post [please submit a PR to fix them 😁](https://github.com/ytmimi/My-Blog)
