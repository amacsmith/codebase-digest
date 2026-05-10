# Publishing your digest to the understand-quickly registry

[`looptech-ai/understand-quickly`](https://github.com/looptech-ai/understand-quickly) is a public, machine-readable registry of code-knowledge and code-context artifacts. It indexes Codebase Digest output through its [`bundle@1`](https://github.com/looptech-ai/understand-quickly/blob/main/schemas/bundle@1.json) format so AI agents (Claude, Codex, Cursor via MCP) can resolve a repo URL to its digest.

Publishing is opt-in. The digest body stays in your repo and is fetched from `raw.githubusercontent.com`; the registry only stores a small JSON pointer.

## One-time setup

1. Register your repo with the registry — `npx @understand-quickly/cli add`, the [wizard](https://looptech-ai.github.io/understand-quickly/add.html), or a manual PR per the [registry CONTRIBUTING.md](https://github.com/looptech-ai/understand-quickly/blob/main/CONTRIBUTING.md).
2. Create a fine-grained GitHub PAT scoped to `looptech-ai/understand-quickly` only with `Repository dispatches: write`. Add it as the `UNDERSTAND_QUICKLY_TOKEN` secret in your repo.

## Workflow

Drop into `.github/workflows/understand-quickly-publish.yml`. It runs `cdigest`, writes a small `bundle@1` sidecar, then hands off to the [`looptech-ai/uq-publish-action`](https://github.com/looptech-ai/uq-publish-action) Marketplace Action.

```yaml
name: understand-quickly publish
on:
  push:
    branches: [main]
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions: { contents: read }
    steps:
      - uses: actions/checkout@v4
      - run: pip install codebase-digest && cdigest . -o markdown -f digest.md
      - name: Build bundle@1 sidecar
        run: |
          python -c "
          import json, os, datetime
          b = os.path.getsize('digest.md')
          json.dump({
            'format': 'bundle@1',
            'manifest': {'tool': 'codebase-digest', 'tool_version': 'latest',
              'generated_at': datetime.datetime.utcnow().isoformat() + 'Z',
              'file_count': 0, 'byte_count': b, 'token_estimate': b // 4,
              'format': 'markdown'},
            'content_url': 'https://raw.githubusercontent.com/${{ github.repository }}/${{ github.ref_name }}/digest.md'
          }, open('digest.bundle.json', 'w'), indent=2)"
      - uses: looptech-ai/uq-publish-action@v0.1.0
        with:
          graph-path: digest.bundle.json
          format: bundle@1
          token: ${{ secrets.UNDERSTAND_QUICKLY_TOKEN }}
```

## Notes

- Submission is opt-in and gated entirely on the secret being set; without `UNDERSTAND_QUICKLY_TOKEN` the Action only stamps metadata locally and exits cleanly.
- See the [producer protocol](https://github.com/looptech-ai/understand-quickly/blob/main/docs/integrations/protocol.md) for the full contract and [`DATA-LICENSE.md`](https://github.com/looptech-ai/understand-quickly/blob/main/DATA-LICENSE.md) for the data-grant semantics.
