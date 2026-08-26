Daily plain-text reports written by `.github/workflows/nightly-report.yml`.

One file per UTC day, `YYYY-MM-DD.txt`. Each reviews every target repo listed
in the `AUTOMATION_REPOS` variable: what ran, what failed, and — the section
worth reading first — which jobs reported success while accomplishing nothing.
