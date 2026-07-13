# Power BI source

Save the report from **Power BI Desktop** into this folder using **File → Save as → Power BI
project (`.pbip`)**, not the binary `.pbix`. The `.pbip` format stores the report and semantic
model as plain text/JSON files (`*.Report/`, `*.SemanticModel/`) that diff cleanly in git and
review well on GitHub.

Build the model by following [`../docs/powerbi-build.md`](../docs/powerbi-build.md).

To publish the live dashboard: open the report in Power BI Desktop and click **Publish** → send
it to a workspace in the Power BI Service (app.powerbi.com). GitHub hosts the *source*; the
Service hosts the *dashboard*.

> A `.pbix` will still commit if you prefer it, but it's binary — no meaningful diffs, and it
> won't render on GitHub. `.gitignore` keeps Power BI's cache/temp files out either way.
