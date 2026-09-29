# hmpps-dpr-incident-reporting-dpds

Data Product Definitions (DPR reports) for the services owned by the Manage Safety team. Despite the
repo name, it is not only for incident reporting.

DPD JSON files live under `definitions/`, one folder per service, and are published to S3 by
[`.github/workflows/publish-dpds.yml`](.github/workflows/publish-dpds.yml).

| Folder | Service |
|---|---|
| `definitions/irs/` | Incident Reporting |

Add a folder when a service gets its first report, for example `non-associations`, `incentives`,
`adjudications`, `csip` or `use-of-force`.

Everything under `definitions/` is published to one S3 folder, `incident-reporting/`, and DPR builds
each report id from that folder name and the file's `id`, not from the file's path. So:

- every published report id starts `incident-reporting_`, whichever service folder it is in
- a file's `id` must be unique across the whole repo, not just within its folder
- moving a file between folders does not change its report id, but the old copy stays in S3 until it
  is deleted (see [Delete](#delete)), leaving two definitions with the same id

## Publish

Actions tab → **Publish DPDs** → Run workflow → pick environment
(`development`, `test`, `preproduction`, `production`).

The reports/dashboards appear in the prisons and probation reporting platforms up to 30 minutes after
publication. The reporting service caches its list of reports for 30 minutes, so a new report returns
an error until that cache expires.

The published report id is the file's `id` prefixed with `incident-reporting_`, e.g.
`incident-reporting_incident-report-live`.

Platform links for the development environment:
https://digital-prison-reporting-mi-ui-dev.hmpps.service.justice.gov.uk/
https://hmpps-probation-mi-ui-dev.hmpps.service.justice.gov.uk

## `experimental/` folders

A service folder can have an `experimental/` subfolder, e.g. `definitions/irs/experimental/`, for
throwaway DPDs used to diagnose problems, usually by stripping parts of a real
definition until a failure goes away. They are published to the lower
environments like anything else, but the publish workflow **excludes them from
production**.

They are not products. Delete them once the investigation that created them is
closed.

There are none at present.

## Live-data reports

The reports with ids ending `-live` read the live Incident Reporting database through Athena, rather
than the reporting datamart, so they are not affected when the datamart falls behind. They are a trial
([IR-2001](https://dsdmoj.atlassian.net/browse/IR-2001)): the original reports in
`hmpps-dpr-data-product-definitions` are unchanged, and the live versions are visible to data wardens
only while the incident reporting leadership team assesses them.

Every live-data report follows these conventions.

| | |
|---|---|
| File | `definitions/irs/<source id>-live.json`, one per source definition |
| Label | Product and report names end "(live data)". Descriptions start "Trial: reads live Incident Reporting data." |
| Access | `INCIDENT_REPORTS__APPROVE` only (the data warden role). Keep the source's row-level caseload policy. |
| Datasource | `athena`: catalog `AwsDataCatalog`, database `reports`, dialect `athena/3` |
| Filter drop-downs | Stay on the `datamart` datasource with their Redshift SQL. DPR looks filter datasets up as a Spring bean, finds no `athena` bean and silently falls back to Redshift, so they cannot run on Athena. |

### Table mapping

Incident Reporting tables are read live. Column names are unchanged.

| Datamart | Live |
|---|---|
| `prisons.incidentreporting_report` | `dps_inc_reporting.public.report` |
| `prisons.incidentreporting_prisoner_involvement` | `dps_inc_reporting.public.prisoner_involvement` |
| `prisons.incidentreporting_question` / `_response` | `dps_inc_reporting.public.question` / `response` |
| `prisons.incidentreporting_constant_*` | `dps_inc_reporting.public.constant_*` |

Reference data is not in the Incident Reporting database, so it stays on the datamart, fully qualified:
`AwsDataCatalog.prisons.prisonregister_prison`, `nomis_agency_locations`, `nomis_areas`,
`nomis_staff_user_accounts`, `nomis_staff_members`, `nomis_offenders`, `nomis_offender_bookings`,
`nomis_agency_internal_locations`.

### Rewriting Redshift SQL for Athena

| Redshift | Athena |
|---|---|
| `LISTAGG(x, ', ') WITHIN GROUP (ORDER BY y)` | `array_join(array_agg(x ORDER BY y), ', ')` |
| correlated `LISTAGG` subquery | grouped `LEFT JOIN` |
| `INITCAP(x)` | `regexp_replace(lower(x), '(^\|[^a-z])([a-z])', m -> m[1] \|\| upper(m[2]))` |
| `x ILIKE y` | `lower(x) LIKE lower(y)` |
| `to_char(ts, 'DD/MM/YYYY')` | `format_datetime(ts, 'dd/MM/yyyy')` |
| `CAST(current_timestamp AS timestamp)` | `format_datetime(current_timestamp AT TIME ZONE 'Europe/London', ...)` (the cast gives UTC) |

### Traps

- Take care joining on the live `uuid` columns (`report.id` and every `report_id`). Athena presents
  them as text, and on an inner join it can push a range filter on them down to Postgres, which
  fails with `operator does not exist: uuid >= character varying`. This happens **even when both
  tables are live**, and only when a filter is selective (one prison, or a caseload), so an
  unfiltered test run can pass and users still hit it. Either use a `LEFT JOIN`, or wrap both sides:
  `ON lower(CAST(r.id AS varchar)) = lower(CAST(pi.report_id AS varchar))`. A plain `CAST` does
  not help, because Athena removes it; `lower()` stops the push-down.
- Policy SQL runs inside Athena, so any table it names must be fully qualified.
- The DPR Tools test rig rejects some keys the schema allows (`metadata.tags`, `dataset.description`,
  `type` on report fields, `wordwrap`). Remove them from the copy you upload to the rig, not from the
  committed file.
- A query Athena rejects leaves nothing in Athena query history. Look in App Insights
  (`exceptions` for `hmpps-dpr-tools-api`) for the reason.

## Validation 

The DPDs are validated against the following schema:
https://raw.githubusercontent.com/ministryofjustice/hmpps-digital-prison-reporting-data-product-definitions-schema/main/schema/1.0.0/data-product-definition-schema.json

This validation is part of the [`.github/workflows/publish-dpds.yml`](.github/workflows/publish-dpds.yml) Workflow.

If you would like to also validate locally you could run:
```sh
npm ci
curl -fsSL \
  https://raw.githubusercontent.com/ministryofjustice/hmpps-digital-prison-reporting-data-product-definitions-schema/main/schema/1.0.0/data-product-definition-schema.json \
  -o schema.json
SCHEMA_LOCATION=$PWD/schema.json npm run validate
```

## Delete
You can delete published DPDs by running the [`.github/workflows/delete-dpds.yml`](.github/workflows/delete-dpds.yml).
This will remove the DPDs from S3, but they will still remain present in the GitHub repo.

`fileNames` takes paths relative to `definitions/`, e.g. `irs/incident-report-live.json`.
