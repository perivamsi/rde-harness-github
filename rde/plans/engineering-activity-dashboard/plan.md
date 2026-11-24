# Plan: engineering-activity-dashboard

<!-- Generated from plan.json, do not edit. Edit the Plan in the Plan View; changes here are lost. -->

- Branch: `rde/early-lobster`
- Writes to: github
- Transform transform:stg_pull_requests (Change c01): Database github
- Transform transform:stg_commits (Change c01): Database github
- Transform transform:stg_pr_review_comments (Change c01): Database github
- Transform transform:stg_issues (Change c01): Database github
- Transform transform:fct_pull_requests (Change c02): Database github
- Transform transform:fct_commits (Change c02): Database github
- Transform transform:fct_pr_review_comments (Change c02): Database github
- Transform transform:fct_issues (Change c02): Database github
- Status: approved
- Approved at 2026-11-24T17:13:53.809Z
- Mode: incremental
- Planned with no guides on base
- Created: 2026-11-24T17:07:47.773Z; updated 2026-11-24T17:13:53.809Z

## Request

> create an engineering activity dashboard based on these tables

## Desire

Engineering leads want one dashboard showing how much work the team is doing and how it flows over the last 90 days, by week: pull requests opened, merged and still open plus median time to merge; commits and active committers; code review comments, reviewers and reviewed PRs; issues opened, closed and still open. Only people count: bot accounts are excluded. It rests on the raw GitHub extract in the github database (raw.pull_requests, raw.commits, raw.pull_request_review_comments, raw.issues), cleaned in staging, modelled as one fact per activity in marts, published to the Library with measures and segments, and read by the dashboard's cards only through those definitions.

## Outcomes

| Id | Outcome | Playbook | Entry | Held constant | Consumption |
|---|---|---|---|---|---|
| eng-activity | An Engineering activity dashboard | deliver | consumption | source-inputs | build-requested |

## Architecture

- **Layers and schemas.** Staging transforms write schema staging, fact transforms write schema marts, both in the github Database; raw stays untouched. There are no guides yet, so this follows the usual raw, staging, marts split and keeps outputs out of the raw source schema.
- **Fact grains.** fct_pull_requests has one row per PR number, fct_commits one per commit sha, fct_pr_review_comments one per review comment id, fct_issues one per issue number excluding PRs. Probes show each key is unique in raw, and every dashboard question is a count or median over one of these activities.
- **Bot rule.** is_bot is derived once in staging (user_type = 'Bot' or login ending in '[bot]'; commits by login only) and excluded through a Human segment per fact on every card. Keeps bot rows available for later analysis while every dashboard number counts people only, as the user chose.
- **Issues exclude pull requests.** stg_issues keeps only rows where is_pull_request is false. GitHub lists PRs in the issues endpoint; 14,033 of the 15,971 raw issue rows are PRs already covered by raw.pull_requests.
- **Window in cards, no dashboard filter.** Each card filters its own date column to the previous 90 days with weekly breakouts; backlog cards show current open counts with no window. Cards use different date columns (created, merged, authored, closed), so one dashboard date filter would be misleading; commits and review comments only exist from Aug 2026.
- **Refresh.** All eight transforms carry one tag run by a daily transform job. Production keeps the dashboard current after merge without manual runs.

## Changes

### staging

#### c01. Add staging transforms for pull requests, commits, review comments and issues, with a daily job (add, eng-activity)

Marts must not read raw, so each raw table gets one staging transform that renames, types and derives is_bot once (architecture: Bot rule; Layers and schemas). No guides exist yet.

Review: accepted

Approach:
1. stg_pull_requests: read raw.pull_requests, one row per number; rename number to pull_request_number, user_login to author_login, draft to is_draft; keep state, base_ref, created_at, closed_at, merged_at (uses `table:github.raw.pull_requests`, `transform:stg_pull_requests`)
2. stg_pull_requests: derive is_merged = merged_at not null; is_bot = user_type = 'Bot' or user_login ends with '[bot]'; write staging.stg_pull_requests (uses `table:github.staging.stg_pull_requests`)
3. stg_commits: read raw.commits, one row per sha as commit_sha; author_login, author_date as authored_at, committer_login, verified as is_verified; is_bot = author_login ends with '[bot]'; write staging.stg_commits (uses `table:github.raw.commits`, `transform:stg_commits`, `table:github.staging.stg_commits`)
4. stg_pr_review_comments: read raw.pull_request_review_comments, one row per id as review_comment_id; parse pull_request_number from pull_request_url after '/pulls/'; user_login as reviewer_login; pull_request_review_id as review_id; created_at (uses `table:github.raw.pull_request_review_comments`, `transform:stg_pr_review_comments`)
5. stg_pr_review_comments: derive is_reply = in_reply_to_id not null; is_bot by the user_type/login rule; write staging.stg_pr_review_comments (uses `table:github.staging.stg_pr_review_comments`)
6. stg_issues: read raw.issues where is_pull_request is false, one row per number as issue_number; title, state, state_reason, user_login as author_login, created_at, closed_at; is_bot by the rule; write staging.stg_issues (uses `table:github.raw.issues`, `transform:stg_issues`, `table:github.staging.stg_issues`)
7. add transform collection Staging (namespace transforms), tag github_activity on all transforms, and job GitHub activity daily at 06:00 running that tag

Files:
- `collections/transforms/staging.yaml`
- `collections/transforms/staging/stg_pull_requests.yaml`
- `collections/transforms/staging/stg_commits.yaml`
- `collections/transforms/staging/stg_pr_review_comments.yaml`
- `collections/transforms/staging/stg_issues.yaml`
- `transforms/transform_tags/github_activity.yaml`
- `transforms/transform_jobs/github_activity_daily.yaml`

Checks:
- stg_pull_requests has 14,031 rows, unique on pull_request_number; 231 have is_bot true
- stg_commits has 2,445 rows, unique on commit_sha; 105 have is_bot true
- stg_pr_review_comments has 9,023 rows, unique on review_comment_id; every pull_request_number parses (0 nulls); 231 have is_bot true
- stg_issues has 1,938 rows (PRs excluded), unique on issue_number; 125 have is_bot true
- no transform uses SELECT *

### marts

#### c02. Add activity facts: fct_pull_requests, fct_commits, fct_pr_review_comments, fct_issues (add, eng-activity)

One fact per activity at the grain each dashboard question counts (architecture: Fact grains), reading staging only (architecture: Layers and schemas).

Review: accepted

Approach:
1. fct_pull_requests: read staging.stg_pull_requests at grain pull_request_number; derive is_open = state 'open', hours_to_merge = (merged_at - created_at) in hours; write marts.fct_pull_requests (uses `table:github.staging.stg_pull_requests`, `transform:fct_pull_requests`, `table:github.marts.fct_pull_requests`)
2. fct_commits: read staging.stg_commits at grain commit_sha, columns carried as they are; write marts.fct_commits (uses `table:github.staging.stg_commits`, `transform:fct_commits`, `table:github.marts.fct_commits`)
3. fct_pr_review_comments: read staging.stg_pr_review_comments at grain review_comment_id; left join staging.stg_pull_requests on pull_request_number for pr_author_login; write marts.fct_pr_review_comments (uses `table:github.staging.stg_pr_review_comments`, `table:github.staging.stg_pull_requests`, `transform:fct_pr_review_comments`, `table:github.marts.fct_pr_review_comments`)
4. fct_issues: read staging.stg_issues at grain issue_number; derive is_open = state 'open', hours_to_close = (closed_at - created_at) in hours; write marts.fct_issues (uses `table:github.staging.stg_issues`, `transform:fct_issues`, `table:github.marts.fct_issues`)
5. add transform collection Marts (namespace transforms); tag every fact github_activity so the daily job runs it after staging

Files:
- `collections/transforms/marts.yaml`
- `collections/transforms/marts/fct_pull_requests.yaml`
- `collections/transforms/marts/fct_commits.yaml`
- `collections/transforms/marts/fct_pr_review_comments.yaml`
- `collections/transforms/marts/fct_issues.yaml`

Checks:
- fct_pull_requests: 14,031 rows, unique pull_request_number; hours_to_merge is null exactly when merged_at is null and never negative
- fct_commits: 2,445 rows, unique commit_sha
- fct_pr_review_comments: 9,023 rows, unique review_comment_id; every pull_request_number exists in fct_pull_requests (0 orphans in raw)
- fct_issues: 1,938 rows, unique issue_number; hours_to_close null when closed_at is null
- planning-time human values over the previous 90 days: 5,727 PRs opened, 4,584 merged, median 21.47 hours to merge, 582 open PRs; 2,312 commits; 8,681 review comments; 761 issues opened, 1,204 open (these move with time)

### semantic

#### c03. Publish the four activity facts to the Library with their keys (add, eng-activity)

Cards read only Library definitions; publishing the facts with key fields makes their measures and segments available and declares the review comment to PR link (architecture: Fact grains).

Review: accepted

Approach:
1. add the github Database file and the Library, Data and Metrics collection files exactly as Metabase defines them
2. publish marts.fct_pull_requests as Pull requests (one row per PR), key pull_request_number (PK) (uses `published-table:github.marts.fct_pull_requests`, `table:github.marts.fct_pull_requests.pull_request_number`)
3. publish marts.fct_commits as Commits (one row per commit), key commit_sha (PK) (uses `published-table:github.marts.fct_commits`, `table:github.marts.fct_commits.commit_sha`)
4. publish marts.fct_pr_review_comments as PR review comments (one row per comment), review_comment_id PK, pull_request_number FK to fct_pull_requests (uses `published-table:github.marts.fct_pr_review_comments`, `table:github.marts.fct_pr_review_comments.review_comment_id`, `table:github.marts.fct_pr_review_comments.pull_request_number`)
5. publish marts.fct_issues as Issues (one row per issue, PRs excluded), key issue_number (PK) (uses `published-table:github.marts.fct_issues`, `table:github.marts.fct_issues.issue_number`)

Files:
- `databases/github/github.yaml`
- `collections/main/library.yaml`
- `collections/main/library/data.yaml`
- `collections/main/library/metrics.yaml`
- `databases/github/schemas/marts/tables/fct_pull_requests/fct_pull_requests.yaml`
- `databases/github/schemas/marts/tables/fct_pull_requests/fields/pull_request_number.yaml`
- `databases/github/schemas/marts/tables/fct_commits/fct_commits.yaml`
- `databases/github/schemas/marts/tables/fct_commits/fields/commit_sha.yaml`
- `databases/github/schemas/marts/tables/fct_pr_review_comments/fct_pr_review_comments.yaml`
- `databases/github/schemas/marts/tables/fct_pr_review_comments/fields/review_comment_id.yaml`
- `databases/github/schemas/marts/tables/fct_pr_review_comments/fields/pull_request_number.yaml`
- `databases/github/schemas/marts/tables/fct_issues/fct_issues.yaml`
- `databases/github/schemas/marts/tables/fct_issues/fields/issue_number.yaml`

Checks:
- the four tables are published in the Library Data collection with display names Pull requests, Commits, PR review comments, Issues
- each table description states its grain and that is_bot marks bot accounts
- fct_pr_review_comments.pull_request_number is a foreign key to fct_pull_requests.pull_request_number

#### c04. Add activity measures and segments, including Human segments that exclude bots (add, eng-activity)

Every dashboard number is defined once in the Library and reused; the Human segments carry the bot exclusion the user chose (architecture: Bot rule).

Review: accepted

Approach:
1. Pull requests: measure PR count = count of rows; measure Median hours to merge = median of hours_to_merge; segments Human PRs = is_bot false, Merged PRs = is_merged true, Open PRs = is_open true (uses `published-table:github.marts.fct_pull_requests`, `measure:pr_count`, `measure:median_hours_to_merge`, `segment:human_prs`, `segment:merged_prs`, `segment:open_prs`)
2. Commits: measure Commit count = count of rows; measure Active committers = distinct author_login; segment Human commits = is_bot false (uses `published-table:github.marts.fct_commits`, `measure:commit_count`, `measure:active_committers`, `segment:human_commits`)
3. PR review comments: measures Review comment count = count, Active reviewers = distinct reviewer_login, Reviewed PRs = distinct pull_request_number; segment Human review comments = is_bot false (uses `published-table:github.marts.fct_pr_review_comments`, `measure:review_comment_count`, `measure:active_reviewers`, `measure:reviewed_prs`, `segment:human_review_comments`)
4. Issues: measure Issue count = count of rows; segments Human issues = is_bot false, Open issues = is_open true, Closed issues = closed_at not null (uses `published-table:github.marts.fct_issues`, `measure:issue_count`, `segment:human_issues`, `segment:open_issues`, `segment:closed_issues`)

Files:
- `databases/github/schemas/marts/tables/fct_pull_requests/measures/pr_count.yaml`
- `databases/github/schemas/marts/tables/fct_pull_requests/measures/median_hours_to_merge.yaml`
- `databases/github/schemas/marts/tables/fct_pull_requests/segments/human_prs.yaml`
- `databases/github/schemas/marts/tables/fct_pull_requests/segments/merged_prs.yaml`
- `databases/github/schemas/marts/tables/fct_pull_requests/segments/open_prs.yaml`
- `databases/github/schemas/marts/tables/fct_commits/measures/commit_count.yaml`
- `databases/github/schemas/marts/tables/fct_commits/measures/active_committers.yaml`
- `databases/github/schemas/marts/tables/fct_commits/segments/human_commits.yaml`
- `databases/github/schemas/marts/tables/fct_pr_review_comments/measures/review_comment_count.yaml`
- `databases/github/schemas/marts/tables/fct_pr_review_comments/measures/active_reviewers.yaml`
- `databases/github/schemas/marts/tables/fct_pr_review_comments/measures/reviewed_prs.yaml`
- `databases/github/schemas/marts/tables/fct_pr_review_comments/segments/human_review_comments.yaml`
- `databases/github/schemas/marts/tables/fct_issues/measures/issue_count.yaml`
- `databases/github/schemas/marts/tables/fct_issues/segments/human_issues.yaml`
- `databases/github/schemas/marts/tables/fct_issues/segments/open_issues.yaml`
- `databases/github/schemas/marts/tables/fct_issues/segments/closed_issues.yaml`

Checks:
- PR count over Human PRs = 13,800 (14,031 minus 231 bot PRs); with Open PRs = 582 at planning time
- Commit count over Human commits = 2,340 (2,445 minus 105 bot commits)
- Review comment count over Human review comments = 8,792 (9,023 minus 231)
- Issue count over Human issues = 1,813 (1,938 minus 125); with Open issues = 1,204 at planning time
- every measure and segment has a description stating its rule

### consumption

#### c05. Add the Engineering activity dashboard (add, eng-activity)

The requested artifact; every card reads a Library measure with its Human segment, previous 90 days by week (architecture: Window in cards, no dashboard filter; Bot rule).

Review: accepted

Approach:
1. PR KPIs, previous 90 days, Human PRs: PRs opened = PR count by created_at; PRs merged = PR count + Merged PRs by merged_at; Median hours to merge = measure + Merged PRs by merged_at; Open PRs = PR count + Open PRs, no window (uses `measure:pr_count`, `measure:median_hours_to_merge`, `segment:human_prs`, `segment:merged_prs`, `segment:open_prs`, `card:engineering_activity/prs_opened`, `card:engineering_activity/prs_merged`, `card:engineering_activity/median_hours_to_merge`, `card:engineering_activity/open_prs`)
2. PR trends: PRs opened per week = PR count by week of created_at; PRs merged per week = PR count + Merged PRs by week of merged_at; both Human PRs, previous 90 days, line charts (uses `measure:pr_count`, `segment:human_prs`, `segment:merged_prs`, `card:engineering_activity/prs_opened_per_week`, `card:engineering_activity/prs_merged_per_week`)
3. Commits, Human commits, previous 90 days by authored_at: Commits per week = Commit count by week (bar); Active committers per week = Active committers by week (line); Top committers = Commit count by author_login, top 10 (row) (uses `measure:commit_count`, `measure:active_committers`, `segment:human_commits`, `card:engineering_activity/commits_per_week`, `card:engineering_activity/active_committers_per_week`, `card:engineering_activity/top_committers`)
4. Code review, Human review comments, previous 90 days by created_at: Review comments per week (bar); Reviewed PRs per week = Reviewed PRs and Active reviewers by week (line); Top reviewers = Review comment count by reviewer_login, top 10 (row) (uses `measure:review_comment_count`, `measure:reviewed_prs`, `measure:active_reviewers`, `segment:human_review_comments`, `card:engineering_activity/review_comments_per_week`, `card:engineering_activity/reviewed_prs_per_week`, `card:engineering_activity/top_reviewers`)
5. Issues, Human issues, previous 90 days: Issues opened per week = Issue count by week of created_at; Issues closed per week = Issue count + Closed issues by week of closed_at; Open issues = Issue count + Open issues, no window (uses `measure:issue_count`, `segment:human_issues`, `segment:closed_issues`, `segment:open_issues`, `card:engineering_activity/issues_opened_per_week`, `card:engineering_activity/issues_closed_per_week`, `card:engineering_activity/open_issues`)
6. dashboard Engineering activity in collection Engineering: heading per section, KPI scalars on top, trends then top-10 tables; description notes bots are excluded and commits/review data start Aug 2026 (uses `dashboard:engineering_activity`)

Files:
- `collections/main/engineering.yaml`
- `collections/main/engineering/engineering_activity.yaml`
- `collections/main/engineering/engineering_activity/prs_opened.yaml`
- `collections/main/engineering/engineering_activity/prs_merged.yaml`
- `collections/main/engineering/engineering_activity/open_prs.yaml`
- `collections/main/engineering/engineering_activity/median_hours_to_merge.yaml`
- `collections/main/engineering/engineering_activity/prs_opened_per_week.yaml`
- `collections/main/engineering/engineering_activity/prs_merged_per_week.yaml`
- `collections/main/engineering/engineering_activity/commits_per_week.yaml`
- `collections/main/engineering/engineering_activity/active_committers_per_week.yaml`
- `collections/main/engineering/engineering_activity/top_committers.yaml`
- `collections/main/engineering/engineering_activity/review_comments_per_week.yaml`
- `collections/main/engineering/engineering_activity/reviewed_prs_per_week.yaml`
- `collections/main/engineering/engineering_activity/top_reviewers.yaml`
- `collections/main/engineering/engineering_activity/issues_opened_per_week.yaml`
- `collections/main/engineering/engineering_activity/issues_closed_per_week.yaml`
- `collections/main/engineering/engineering_activity/open_issues.yaml`

Checks:
- every card returns rows on the branch and aggregates a Library measure (no native SQL, no staging or raw table)
- every card applies its table's Human segment
- scalars at planning time: PRs opened 5,727, PRs merged 4,584, median hours to merge 21.47, open PRs 582, open issues 1,204 (they move with time)
- dashboard has four sections: Pull request flow, Commits, Code review, Issues
