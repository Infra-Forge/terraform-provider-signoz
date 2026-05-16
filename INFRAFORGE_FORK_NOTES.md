# Infra-Forge fork notes

This fork of `signoz/terraform-provider-signoz` exists to carry **two kinds of changes**:

1. Bug fixes that block our internal stack and have not yet shipped upstream.
2. New SigNoz resources we need before upstream supports them.

Once a change lands upstream with a comparable fix, the corresponding patch
in this fork gets reverted and we move closer to retiring the fork entirely.

## Current delta vs upstream

- `signoz/internal/model/dashboard.go` — empty-string handling on
  `SetVariables`, `SetLayout`, `SetWidgets`, plus reader-side normalization
  on `VariablesToTerraform` and `PanelMapToTerraform` so `""` round-trips
  through Create/Update/Read cleanly. Released as `v0.0.12`.

## Releasing a new fork version

```shell
# 1. Make code changes on a branch off main, run go test ./... if applicable.
# 2. Bump the version (plain SemVer; pre-release suffixes like
#    "0.0.12-infraforge.1" confuse Terraform's required_providers parser).
VERSION=0.0.13
git tag -a "v${VERSION}" -m "Fork v${VERSION}: <one-line summary>"

# 3. Build cross-platform binaries (omit darwin_amd64 unless someone needs it).
rm -rf dist && mkdir -p dist
for goos in darwin linux; do
  for goarch in amd64 arm64; do
    [ "$goos" = "darwin" ] && [ "$goarch" = "amd64" ] && continue
    GOOS=$goos GOARCH=$goarch \
      go build -ldflags="-s -w -X main.version=v${VERSION}" \
        -o "dist/terraform-provider-signoz_v${VERSION}_${goos}_${goarch}" ./
  done
done
( cd dist && shasum -a 256 * > "terraform-provider-signoz_v${VERSION}_SHA256SUMS" )

# 4. Push the tag and cut a release with the binaries attached.
git push origin "v${VERSION}"
gh release create "v${VERSION}" \
  --repo Infra-Forge/terraform-provider-signoz \
  --title "v${VERSION} — <summary>" \
  --notes "..." \
  dist/*

# 5. Bump SIGNOZ_PROVIDER_VERSION in infranotes-terraform/Makefile and
#    .github/workflows/terraform-signoz.yaml, plus modules/signoz-observability/versions.tf
#    and stacks/signoz/versions.tf. Run `make install-signoz-provider` then
#    `terraform init` + `terraform plan` to validate locally.
```

## Adding a new resource (= more API coverage)

The upstream provider follows a strict layout: the SigNoz HTTP client lives in
`signoz/internal/client/`, resource models in `signoz/internal/model/`, the
Terraform-plugin-framework resource code in `signoz/internal/provider/resource/`,
and attribute names in `signoz/internal/attr/`.

To add (for example) a `signoz_notification_channel` resource:

1. **HTTP client** — `signoz/internal/client/notification_channel.go`. Follow the
   shape of `alert.go` / `dashboard.go`: a typed `GetX` / `CreateX` / `UpdateX` /
   `DeleteX` taking `(ctx, payload)` and returning the parsed model. Hit the
   matching SigNoz API path (look it up in
   [SigNoz docs](https://signoz.io/docs/) or in the SigNoz frontend's network
   tab against a real cluster). Use `doRequest` for the actual HTTP call.

2. **Model** — `signoz/internal/model/notification_channel.go`. Plain Go struct
   with `json:` tags matching the API payload. Add `ToTerraform*` helpers for
   any fields that need TF state coercion (lists, JSON strings, etc.) and
   `Set*` methods for the inverse direction. Always guard string-encoded JSON
   setters against `IsNull` and `ValueString() == ""` (see the dashboard fork
   diff for the pattern).

3. **Attribute names** — `signoz/internal/attr/notification_channel.go`. One
   constant per field; the resource schema and the model JSON tags both
   reference these so renames stay safe.

4. **Resource** — `signoz/internal/provider/resource/notification_channel.go`.
   Implement `resource.Resource`, `resource.ResourceWithConfigure`, and
   (recommended) `resource.ResourceWithImportState`. Look at
   `dashboard.go` for the shape; mirror its `Metadata` / `Schema` / `Create` /
   `Read` / `Update` / `Delete` / `ImportState` methods.

5. **Provider wiring** — `signoz/internal/provider/provider.go`. Add the new
   resource constructor to the `Resources` slice. Also add it to the
   datasource list if a `signoz_notification_channel` datasource is useful.

6. **Docs** — `templates/resources/notification_channel.md.tmpl` and run
   `go generate ./...` to regenerate `docs/`. Upstream uses
   `github.com/hashicorp/terraform-plugin-docs`.

7. **Examples** — `examples/resources/signoz_notification_channel/resource.tf`
   with the minimum-viable invocation.

8. **Tests** — add a basic acceptance test under `signoz/internal/provider/resource/`.
   Upstream's existing tests show the pattern; mark them with the standard
   `TestAcc` build tag.

9. **Release** — see the previous section.

## Eventually upstreaming

For each fork-only change, open an upstream issue and reference the
Infra-Forge release that carries the fix. If upstream merges a comparable
fix, drop the local patch and revert the version pin.
