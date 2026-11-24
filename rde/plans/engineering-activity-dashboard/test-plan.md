# Test Plan: engineering-activity-dashboard

<!-- Generated from test-plan.json, do not edit. -->

## An Engineering activity dashboard

| Test | Kind | Invariant | Task | Proves |
|---|---|---|---|---|
| stg_pull_requests derives is_bot and is_merged (`tt-stg-pull-requests`) | transform |  | t-staging-activity | Pins the PR bot rule (user type or [bot] login, nulls not bot) and is_merged. |
| stg_commits flags bot authors by login and keeps every commit (`tt-stg-commits`) | transform |  | t-staging-activity | Pins the commit bot rule by login and that null-login commits are kept. |
| stg_pr_review_comments parses the PR number and flags replies and bots (`tt-stg-pr-review-comments`) | transform |  | t-staging-activity | Pins PR-number parsing from the URL, reply and bot flags. |
| stg_issues drops pull requests and flags bot authors (`tt-stg-issues`) | transform |  | t-staging-activity | Pins that pull requests are dropped from issues and null is_pull_request rows kept. |
| stg_pull_requests: pull_request_number is unique (`dc-stg-pull-requests-unique`) | data-check |  | t-staging-activity | Grain of stg_pull_requests. |
| stg_pull_requests: pull_request_number is present (`dc-stg-pull-requests-key-present`) | data-check |  | t-staging-activity | Key of stg_pull_requests is present. |
| stg_pull_requests keeps every raw pull request (`dc-stg-pull-requests-rows`) | data-check |  | t-staging-activity | No raw PR is lost in staging. |
| stg_commits: commit_sha is unique (`dc-stg-commits-unique`) | data-check |  | t-staging-activity | Grain of stg_commits. |
| stg_commits: commit_sha is present (`dc-stg-commits-key-present`) | data-check |  | t-staging-activity | Key of stg_commits is present. |
| stg_commits keeps every raw commit (`dc-stg-commits-rows`) | data-check |  | t-staging-activity | No raw commit is lost in staging. |
| stg_pr_review_comments: review_comment_id is unique (`dc-stg-pr-review-comments-unique`) | data-check |  | t-staging-activity | Grain of stg_pr_review_comments. |
| stg_pr_review_comments: review_comment_id and pull_request_number are present (`dc-stg-pr-review-comments-key-present`) | data-check |  | t-staging-activity | Every review comment has its key and a parsed PR number. |
| stg_pr_review_comments keeps every raw review comment (`dc-stg-pr-review-comments-rows`) | data-check |  | t-staging-activity | No raw review comment is lost in staging. |
| stg_issues: issue_number is unique (`dc-stg-issues-unique`) | data-check |  | t-staging-activity | Grain of stg_issues. |
| stg_issues: issue_number is present (`dc-stg-issues-key-present`) | data-check |  | t-staging-activity | Key of stg_issues is present. |
| stg_issues has fewer rows than raw.issues (pull requests removed) (`dc-stg-issues-fewer-than-raw`) | data-check |  | t-staging-activity | PR rows are filtered out of issues. |
| fct_pull_requests derives is_open and hours_to_merge (`tt-fct-pull-requests`) | transform |  | t-marts-activity-facts | Pins is_open and hours_to_merge, including sub-hour merges and unmerged PRs. |
| fct_commits keeps every commit with its author and bot flag (`tt-fct-commits`) | transform |  | t-marts-activity-facts | Pins that every commit passes to the fact. |
| fct_pr_review_comments adds the PR author without losing or duplicating comments (`tt-fct-pr-review-comments`) | transform |  | t-marts-activity-facts | Pins the PR-author join: no fan-out, no lost comments. |
| fct_issues derives is_open and hours_to_close (`tt-fct-issues`) | transform |  | t-marts-activity-facts | Pins is_open and hours_to_close. |
| fct_pull_requests: pull_request_number is unique (`dc-fct-pull-requests-unique`) | data-check |  | t-marts-activity-facts | Grain of fct_pull_requests. |
| fct_pull_requests: pull_request_number is present (`dc-fct-pull-requests-key-present`) | data-check |  | t-marts-activity-facts | Key of fct_pull_requests is present. |
| fct_pull_requests has one row per staged pull request (`dc-fct-pull-requests-rows`) | data-check |  | t-marts-activity-facts | fct_pull_requests matches staging row for row. |
| fct_pull_requests: hours_to_merge is never negative (heuristic) (`dc-fct-pull-requests-hours-non-negative`) | data-check |  | t-marts-activity-facts | Time to merge is never negative. |
| fct_commits: commit_sha is unique (`dc-fct-commits-unique`) | data-check |  | t-marts-activity-facts | Grain of fct_commits. |
| fct_commits: commit_sha is present (`dc-fct-commits-key-present`) | data-check |  | t-marts-activity-facts | Key of fct_commits is present. |
| fct_commits has one row per staged commit (`dc-fct-commits-rows`) | data-check |  | t-marts-activity-facts | fct_commits matches staging row for row. |
| fct_pr_review_comments: review_comment_id is unique (`dc-fct-pr-review-comments-unique`) | data-check |  | t-marts-activity-facts | Grain of fct_pr_review_comments. |
| fct_pr_review_comments: review_comment_id is present (`dc-fct-pr-review-comments-key-present`) | data-check |  | t-marts-activity-facts | Key of fct_pr_review_comments is present. |
| fct_pr_review_comments has one row per staged review comment (`dc-fct-pr-review-comments-rows`) | data-check |  | t-marts-activity-facts | fct_pr_review_comments matches staging row for row. |
| every review comment's pull_request_number is a fct_pull_requests key (`dc-fct-pr-review-comments-pr-ref`) | data-check |  | t-marts-activity-facts | Every review comment links to a known pull request. |
| fct_issues: issue_number is unique (`dc-fct-issues-unique`) | data-check |  | t-marts-activity-facts | Grain of fct_issues. |
| fct_issues: issue_number is present (`dc-fct-issues-key-present`) | data-check |  | t-marts-activity-facts | Key of fct_issues is present. |
| fct_issues has one row per staged issue (`dc-fct-issues-rows`) | data-check |  | t-marts-activity-facts | fct_issues matches staging row for row. |
| fct_issues: hours_to_close is never negative (heuristic) (`dc-fct-issues-hours-non-negative`) | data-check |  | t-marts-activity-facts | Time to close is never negative. |
| PR count over Human PRs equals the probed 13,800 (`sa-human-pr-count`) | semantic |  | t-semantic-measures | PR count with the Human segment returns the probed people-only total. |
| Commit count over Human commits equals the probed 2,340 (`sa-human-commit-count`) | semantic |  | t-semantic-measures | Commit count with the Human segment returns the probed total. |
| Review comment count over Human review comments equals the probed 8,792 (`sa-human-review-comment-count`) | semantic |  | t-semantic-measures | Review comment count with the Human segment returns the probed total. |
| Issue count over Human issues equals the probed 1,813 (`sa-human-issue-count`) | semantic |  | t-semantic-measures | Issue count with the Human segment returns the probed total. |
| PR count over Human PRs and Open PRs equals the probed 582 (`sa-human-open-prs`) | semantic |  | t-semantic-measures | Open PRs backlog definition returns the probed count. |
| Median hours to merge over human merged PRs equals the probed 26.44 (`sa-human-median-hours-to-merge`) | semantic |  | t-semantic-measures | Median hours to merge matches the probed median. |
| Reviewed PRs over Human review comments equals the probed 1,247 (`sa-human-reviewed-prs`) | semantic |  | t-semantic-measures | Reviewed PRs distinct count matches the probe. |
| PRs opened resolves to PR count and returns data (`ca-prs-opened`) | consumption |  | t-consumption-dashboard | Dashboard card exists, reads PR count, returns data. |
| PRs merged resolves to PR count and returns data (`ca-prs-merged`) | consumption |  | t-consumption-dashboard | Dashboard card exists, reads PR count, returns data. |
| Open PRs resolves to PR count and returns data (`ca-open-prs`) | consumption |  | t-consumption-dashboard | Dashboard card exists, reads PR count, returns data. |
| Median hours to merge card resolves to its measure and returns data (`ca-median-hours-to-merge`) | consumption |  | t-consumption-dashboard | Dashboard card exists, reads the median measure, returns data. |
| PRs opened per week resolves to PR count and returns data (`ca-prs-opened-per-week`) | consumption |  | t-consumption-dashboard | Dashboard trend exists, reads PR count, returns data. |
| PRs merged per week resolves to PR count and returns data (`ca-prs-merged-per-week`) | consumption |  | t-consumption-dashboard | Dashboard trend exists, reads PR count, returns data. |
| Commits per week resolves to Commit count and returns data (`ca-commits-per-week`) | consumption |  | t-consumption-dashboard | Dashboard trend exists, reads Commit count, returns data. |
| Active committers per week resolves to Active committers and returns data (`ca-active-committers-per-week`) | consumption |  | t-consumption-dashboard | Dashboard trend exists, reads Active committers, returns data. |
| Top committers resolves to Commit count and returns data (`ca-top-committers`) | consumption |  | t-consumption-dashboard | Top committers exists, reads Commit count, returns data. |
| Review comments per week resolves to Review comment count and returns data (`ca-review-comments-per-week`) | consumption |  | t-consumption-dashboard | Dashboard trend exists, reads Review comment count, returns data. |
| Reviewed PRs per week resolves to Reviewed PRs and returns data (`ca-reviewed-prs-per-week`) | consumption |  | t-consumption-dashboard | Dashboard trend exists, reads Reviewed PRs, returns data. |
| Top reviewers resolves to Review comment count and returns data (`ca-top-reviewers`) | consumption |  | t-consumption-dashboard | Top reviewers exists, reads Review comment count, returns data. |
| Issues opened per week resolves to Issue count and returns data (`ca-issues-opened-per-week`) | consumption |  | t-consumption-dashboard | Dashboard trend exists, reads Issue count, returns data. |
| Issues closed per week resolves to Issue count and returns data (`ca-issues-closed-per-week`) | consumption |  | t-consumption-dashboard | Dashboard trend exists, reads Issue count, returns data. |
| Open issues resolves to Issue count and returns data (`ca-open-issues`) | consumption |  | t-consumption-dashboard | Dashboard card exists, reads Issue count, returns data. |
