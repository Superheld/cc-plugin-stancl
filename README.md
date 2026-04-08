# cc-plugin-stancl

A [Claude Code](https://claude.com/claude-code) plugin that teaches Claude how to work with the [stancl/*](https://github.com/stancl) Laravel ecosystem.

Once installed, Claude automatically picks up the right skill whenever you work with one of these packages in a Laravel project — no manual prompting needed.

## Included skills

- **tenancy** — [stancl/tenancy](https://github.com/stancl/tenancy): multi-tenancy setup, tenant models, `BelongsToTenant`, bootstrappers, tenant-aware routes/jobs/migrations, isolation testing.
- **jobpipeline** — [stancl/jobpipeline](https://github.com/stancl/jobpipeline): turning jobs into chained event listeners with `JobPipeline::make`.
- **virtualcolumn** — [stancl/virtualcolumn](https://github.com/stancl/virtualcolumn): schemaless attributes on Eloquent models via the `VirtualColumn` trait.

## Installation

In Claude Code, add this repo as a marketplace and install the plugin:

```
/plugin marketplace add Superheld/cc-plugin-stancl
/plugin install cc-plugin-stancl@cc-plugin-stancl
```

## Usage

Just ask Claude about anything stancl-related — for example:

- *"Set up multi-tenancy with single-database mode"*
- *"Add `BelongsToTenant` to the Order model"*
- *"Convert this job into a JobPipeline event listener"*
- *"Store `settings` as a virtual column on the User model"*

Claude will load the matching skill automatically and follow the conventions documented inside it.

## License

MIT
