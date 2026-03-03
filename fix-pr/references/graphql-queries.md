# GraphQL Queries Reference

## Fetch unresolved review threads (paginated)

```graphql
query($cursor: String) {
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 100, after: $cursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          isResolved
          line
          path
          comments(first: 3) {
            nodes {
              body
              path
              author { login }
              createdAt
            }
          }
        }
      }
    }
  }
}
```

## Count unresolved threads

```bash
gh api graphql -f query='...' --jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false)] | length'
```

## Filter to Macroscope comments only

```bash
gh api graphql -f query='...' --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false) | select(.comments.nodes[0].author.login == "macroscopeapp[bot]")'
```

## Batch-resolve threads

```graphql
mutation {
  t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: "ID2"}) { thread { isResolved } }
  t3: resolveReviewThread(input: {threadId: "ID3"}) { thread { isResolved } }
}
```

## Check Macroscope status

```bash
gh pr view --json statusCheckRollup --jq '.statusCheckRollup[] | select(.name | test("Macroscope")) | "\(.status) \(.conclusion)"'
```
