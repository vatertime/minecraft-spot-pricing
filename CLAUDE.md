# CLAUDE.md

CloudFormation template that runs the [itzg/docker-minecraft-server](https://github.com/itzg/docker-minecraft-server) image on ECS, on a single EC2 instance (spot or on demand) in an Auto Scaling Group, with the world on EFS. Forked from [m-chandler/factorio-spot-pricing](https://github.com/m-chandler/factorio-spot-pricing).

## Files

- `cf.yml` - the whole stack. Users upload this file in the CloudFormation console, so it must stay a single self-contained template (no nested stacks, macros or build step).
- `readme.md` - user-facing docs, including Expected Costs and a Changelog.

Both files use CRLF line endings. Preserve them when editing.

## Checks

There is no CI and no test suite. Before pushing, run:

```bash
pip install cfn-lint   # once
cfn-lint cf.yml
```

Changes can only be truly verified by deploying a stack in AWS, which isn't possible from a dev container. Say so in the PR when a change hasn't been deployed.

## Conventions

- **Exposing an itzg env var**: add a parameter (with a `Description`), a `<Name>Provided` condition, an entry in `Metadata` (ParameterGroups and ParameterLabels), and an `!If` entry in the container's `Environment` list. Copy the `Whitelist` blocks. Check the variable against the current itzg docs (https://docker-minecraft-server.readthedocs.io/) rather than memory, since the image changes often.
- **Existing stacks**: on a stack update CloudFormation keeps each parameter's previous value, so changing a `Default` only affects new stacks. Don't rename or remove parameters without a reason, because that breaks updates and the readme's start/stop script. Note behaviour changes for existing stacks in the readme Changelog.
- **Readme**: update it in the same PR as the template. Parameter names, ECS cluster/service names (`<stack>-cluster`, `<stack>-ecs-service`) and cost figures in the readme must match `cf.yml`.
- **Upstream**: the Factorio template is the upstream. Ported changes are worth noting in the Changelog as "Ported from upstream".
