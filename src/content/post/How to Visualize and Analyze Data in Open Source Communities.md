---
title: "How to Visualize and Analyze Data in Open Source Communities"
description: "This article will introduce the development approach and implementation of OPENALYSIS."
publishDate: "21 Apr 2024"
tags: ["opensource", "go", "programming", "graphql"]
---

## Introduction

This article will introduce the development approach and implementation of [OPENALYSIS](https://github.com/B1NARY-GR0UP/openalysis), an open-source community data analysis service. We will first preview the effects of the service after deployment, and then delve into the entire project's development process from basic ideas to concrete implementation.

## Demonstration of Results

After successful deployment, the data display panel looks as follows. Here, we present statistics and displays of the open-source community [CloudWeGo](https://www.cloudwego.io/), a project by ByteDance. The panels include:

- Key data statistics, including: total number of Issues, total number of Pull Requests (PRs), total number of Forks, total number of Stars, and total number of Contributors (deduplicated).
- Ranking of all contributors based on their contributions.
- Distribution of contributors by region and company.
- Number of Issues and PRs initiated by contributors.
- Number of days since contributors' first PR.
- List and quantity of projects contributors have participated in.
- Assignees of Issues and PRs across all repositories (Open Issues, Open PRs).
- Trends in Stars, Forks, Issues, PRs, and Contributors for major repositories.

![example-1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cwmuy9z2v4m5ztif0ni3.png)

![example-2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3esxjjyngj3apupftc8v.png)

![example-3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/l390f804df1cooyg9xcr.png)

![example-4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/emw1l3zbbrjd9z2gfer8.png)

## Basic Approach

The basic approach of this project is to retrieve corresponding data through the API provided by GitHub, persist the retrieved data into a database, and finally display the data stored in the database through Grafana Dashboard. Additionally, it requires using scheduled tasks to periodically fetch the latest data and update the persisted data.

- **Data Acquisition:** GitHub provides two different types of APIs, [REST (v3)](https://docs.github.com/en/rest?apiVersion=2022-11-28) and [GraphQL (v4)](https://docs.github.com/en/graphql). We primarily utilize the **GraphQL API** for data retrieval, resorting to the REST API when fetching data not supported by v4.

  GraphQL API offers more flexibility in obtaining the data we need compared to REST API. [This example](https://docs.github.com/en/graphql/guides/migrating-from-rest-to-graphql#example-getting-the-data-you-need-and-nothing-more) vividly illustrates the advantages of GraphQL over REST.

- **Data Storage:** We use the most common **MySQL** database to persistently store our data.

- **Data Analysis:** We employ **Grafana Dashboard** to query the MySQL database and display the data by configuring different types of panels.

## Implementation Approach

### Concept of Groups

Given that the repositories within the CloudWeGo community span multiple GitHub organizations and include separate repositories, such as:

- https://github.com/cloudwego (org)
- https://github.com/kitex-contrib (org)
- https://github.com/hertz-contrib (org)
- https://github.com/volo-rs (org)
- https://github.com/bytedance/sonic (repo)
- https://github.com/bytedance/monoio (repo)

If we aim to gather data for the entire community, we need to establish a new level of granularity to encompass these organizations and repositories.

In our project, we introduce the concept of **Groups**. A group can consist of multiple organizations or repositories. This way, we can consider the entire CloudWeGo community as a group and perform data retrieval, storage, analysis, etc., at the group level.

### Data Acquisition

As we primarily utilize the GitHub GraphQL API (v4) for data retrieval, we can craft the corresponding GraphQL query based on the data we wish to obtain, leveraging the [Public schema](https://docs.github.com/en/graphql/overview/public-schema) provided by GitHub. Subsequently, we send a request to GitHub's GraphQL endpoint at https://api.github.com/graphql.

For instance, if we aim to retrieve the number of issues, pull requests (PRs), stars, and forks for a repository, the corresponding GraphQL query would be as follows:

```graphql
query RepoInfo($owner: String!, $name: String!) {
    repository(owner: $owner, name: $name) {
        id
        owner {
            id
        }
        issues {
            totalCount
        }
        pullRequests {
            totalCount
        }
        stargazers {
            totalCount
        }
        forks {
            totalCount
        }
    }
}
```

The `$owner` and `$name` are parameters required for this query, which can be passed in the form of Variables:

```json
{
    "owner": "cloudwego",
    "name": "hertz"
}
```

However, this simple single query is not sufficient to fulfill all data retrieval needs.

**GitHub limits the number of items that can be obtained through a single query via the GraphQL API to 100, in order to prevent users from making excessive or abusive requests to GitHub servers.**

For example, if we want to retrieve a list of names of all repositories under an organization, but the total number of repositories owned by this organization is 150, then in a single GraphQL query, we can only obtain the names of up to 100 repositories, leaving the names of the remaining 50 repositories to be obtained in the next query. (This might not be a very good example because there are rarely organizations with more than 100 repositories, but you can imagine a query to retrieve all issues of repositories, and most large projects have more than 100 issues.)

To handle cases where a single query cannot retrieve all the data, GitHub GraphQL API provides **Pagination**. By using Pagination, we can make a series of consecutive requests to obtain the complete data.

Continuing with the example above, a GraphQL query to obtain a list of names of all repositories under an organization using Pagination would be:

```graphql
query QueryReposByOrg($login: String!, $first: Int!, $after: String) {
    organization(login: $login) {
        repositories(first: $first, after: $after) {
            totalCount
            pageInfo {
                hasNextPage
                endCursor
            }
            nodes {
                nameWithOwner
            }
        }
    }
}
```

The `$login`, `$first`, and `$after` are parameters required for this query. `$first` represents the number of entries to retrieve in a single query, while `$after` serves as a cursor. By setting the cursor, entries can be retrieved starting from a specific position. Here, it retrieves `$first` number of entries starting from the position indicated by `$after`.

The Variables for the first query are configured as follows. Since we don't know the value of the cursor when initiating the first query, it is not set:

```json
{
    "login": "cloudwego",
    "first": 100,
    "after": null
}
```

After the first query is completed, we can use the values of `hasNextPage` and `endCursor` from the returned `pageInfo` to initiate subsequent requests. For instance, when we determine that the value of `hasNextPage` is `true`, it indicates that there are still data entries to be retrieved. In this case, we only need to pass the value of `endCursor` as a parameter to the `$after` field of the next query. This way, the next query will start retrieving data entries from the last one obtained in the current query.

In terms of code implementation, we can use a `for` loop to carry out this series of requests using pagination. Below is the Go language implementation of the example mentioned above:

```go
type RepoName struct {
	Organization struct {
		Repositories struct {
			PageInfo struct {
				HasNextPage bool
				EndCursor   string
			}
			Nodes []struct {
				NameWithOwner string
			}
		} `graphql:"repositories(first: $first, after: $after)"`
	} `graphql:"organization(login: $login)"`
}

// QueryRepoNameByOrg return repos of the provided org in `org/repo` format
func QueryRepoNameByOrg(ctx context.Context, login string) ([]string, error) {
	query := &RepoName{}
	variables := map[string]interface{}{
		"login": githubv4.String(login),
		"first": githubv4.Int(100),
		"after": (*githubv4.String)(nil),
	}
	var (
		repos []struct {
			NameWithOwner string
		}
		names []string
	)
	for {
		if err := GlobalV4Client.Query(ctx, query, variables); err != nil {
			return nil, err
		}
		repos = append(repos, query.Organization.Repositories.Nodes...)
		if !query.Organization.Repositories.PageInfo.HasNextPage {
			break
		}
		variables["after"] = githubv4.NewString(githubv4.String(query.Organization.Repositories.PageInfo.EndCursor))
	}
	for _, repo := range repos {
		names = append(names, repo.NameWithOwner)
	}
	return names, nil
}
```

Through pagination, we can handle most of the data retrieval needs.

> Here are some pitfalls to watch out for:
>
> - `issues` have a `since` field, but `pullrequests` do not. Therefore, it's easy to retrieve all `issues` updated after a certain timestamp by using the `since` field from the last retrieval. However, for `pullrequests`, you can only retrieve those updated after the last queried cursor is saved.
> - When you initiate a query using a cursor, but there are no data updates, the returned cursor is empty instead of the cursor value you used for the query.
> - Some data can only be retrieved through the REST API.

### Data Storage and Scheduled Tasks

All tasks needing execution within the project are divided into `InitTask` and `UpdateTask`. The `InitTask` is used to fetch and store the complete data of orgs and repos included in the configured groups upon project startup, while the `UpdateTask` is used for scheduled tasks after project initiation to fetch and store data updated since the last retrieval.

For instance, within `InitTask`, it's necessary to retrieve all issues from a repository, whereas within `UpdateTask`, only new issues or those updated since the last retrieval need to be fetched from a repository.

In `UpdateTask`, besides updating a portion of old data in the database, there is also a portion of data that doesn't require updating. This is because we need to retain historical data to form a **time series**, which enables the construction of graphs depicting the trend of data changes in subsequent data analyses.

Below is a simple example of a one-dimensional time series, illustrating temperature variations at two locations, LGA and BOS:

| StartTime | Temp | Location |
| :-------- | :--- | :------- |
| 09:00     | 24   | LGA      |
| 09:00     | 20   | BOS      |
| 10:00     | 26   | LGA      |
| 10:00     | 22   | BOS      |

In our specific project, we need to store the data of each repository as time series. Therefore, in each `UpdateTask`, we need to retrieve and store data such as the number of issues, stars, etc., for all repositories.

By storing the time series, we can easily generate trend graphs like the one below:

![example-5](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/82sh01dy9ub3idnq2jq1.png)

Another important point to note is that when storing data into MySQL, it's necessary to encapsulate a complete operation within a transaction. This ensures that if an error occurs during data retrieval due to network issues or other reasons, we can roll back the entire transaction and retry the operation. Alternatively, if you're conducting a larger operation, you can roll back a portion of the transaction and retry.

### Data Analysis

Once we've fetched data through the GitHub GraphQL API and stored it in MySQL, we can then utilize Grafana for data analysis and visualization.

Grafana offers a variety of visualization options, making it convenient to select based on the type and format of the stored data. For instance, data stored in a time series format, as mentioned earlier for various repositories, can be visualized using Grafana's `Time series` visualization. On the other hand, for displaying community contributors' companies and geographical distribution, we can utilize the `Pie chart` visualization.

![example-6](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jjg3dm29tlp93w9ziuwr.png)

After selecting the appropriate visualization method, we can proceed with querying the data and displaying it. Since we use MySQL as our data persistence mechanism, our primary method of data querying is SQL. By writing different SQL queries to retrieve the corresponding data in various ways, coupled with the multiple visualization options provided by Grafana, we can easily achieve the desired effects showcased at the beginning of the article.

For instance, the following is a visualization panel of the community contribution leaderboard:

![example-7](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xd05wmacv87bqu66162b.png)

The corresponding SQL is shown below. We arrange it in descending order based on the contributions statistics of each contributor. By utilizing the `DENSE_RANK()` function, we ensure that rankings are assigned consecutively when dealing with contributors who have the same contributions:

```sql
SELECT
    DENSE_RANK() OVER (ORDER BY SUM(contributions) DESC) AS 'Rank',
    login AS 'Name',
    SUM(contributions) AS 'Contributions'
FROM
    openalysis.contributors
GROUP BY
    login
ORDER BY
    SUM(contributions) DESC;
```

## Summary

That concludes all the content of this article. We started from showcasing the final results and then delved into the specific processes of transforming the basic ideas into implementations, gaining an understanding of the entire project development.

OPENALYSIS still has many shortcomings, such as not providing more scalable Grafana Dashboards, which will be addressed in future development efforts.

We hope this article is helpful to readers. If there are any mistakes or questions, feel free to comment or reach out via private message.

## References

- https://github.com/B1NARY-GR0UP/openalysis
- https://www.cloudwego.io

- https://docs.github.com/en