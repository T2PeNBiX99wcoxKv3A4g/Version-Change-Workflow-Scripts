# Version Change Workflow Scripts

Helpers used by my GitHub workflows to bump versions and keep secondary version files in sync.
They are not C#-specific: any text-based version file is supported.

## change_version.py

| Command               | Status     | Purpose                                                              |
|-----------------------|------------|----------------------------------------------------------------------|
| `set-version`         | current    | Set the authoritative version in the primary file and record old/new |
| `sync-version`        | current    | Apply the recorded old → new version to another file                 |
| `create-version-file` | current    | Write `.version` only, when the new version comes from elsewhere     |
| `change-version`      | deprecated | Alias for `set-version`                                              |
| `replace-version`     | deprecated | Alias for `sync-version`                                             |

Call `set-version` **once** per release, then `sync-version` for every other file. Re-running
`set-version` with the version already in the file is a successful no-op, so workflow retries are
safe; use `sync-version` to update secondary files.

### `set-version`

Detects the current version in a file, replaces it with a new one, and records the pair in
`./.version` and `./.version_old` for `sync-version`.

```shell
python change_version.py set-version <file_path> <version_key> <new_version> [OPTIONS]
```

| Option               | Description                                                |
|----------------------|------------------------------------------------------------|
| `--require-keywords` | Comma-separated keywords that the target line must contain |
| `--debug`            | Enable `icecream` debug output                             |

`version_key` is a complete key name, matched case-sensitively, so `Version`
never matches `AssemblyVersion` or `<SupportedOSPlatformVersion>`. The value
follows a separator (`=`, `:`, `>` or `(`) with an optional surrounding quote, so
`version = "1.2.3"`, `"Version": "1.2.3"` and `<Version>1.2.3</Version>` all work.
Lines that start with a comment marker (`#`, `!`, `//`, `<!--`, `;`) are skipped.

The version is detected on the first matching line and then replaced **only on
lines whose key matches**, so no other line is touched. Every matching line is a
target, which handles a version that lives in several places at once. A matching
key whose value is a *different* version is left alone, so `Version="10.0.13"` in
a `PackageReference` is not clobbered when the project version is `0.1.0`.

If no line is replaced because the file already holds the requested version, the command is a
successful no-op and still records `./.version` and `./.version_old` for `sync-version`. A different
version that cannot be replaced is still an error.

```shell
# csproj:            <Version>0.1.0</Version>
python change_version.py set-version App.csproj Version 0.2.0

# JVM properties:    version=0.1.0
python change_version.py set-version gradle.properties version 0.2.0

# thunderstore.toml: versionNumber = "0.1.0"
python change_version.py set-version thunderstore.toml versionNumber 0.2.0

# JSON:              "Version": "0.1.0"
python change_version.py set-version manifest.json Version 0.2.0

# C#:                [assembly: AssemblyVersion("0.1.0.0")]
python change_version.py set-version Properties/AssemblyInfo.cs AssemblyVersion 0.2.0.0
```

### `sync-version`

`set-version` writes the detected old version to `./.version_old` and the new one to `./.version`,
both in the current working directory. `sync-version` reuses that pair to update any other file,
which keeps secondary files in sync. When the recorded old and new versions are equal, it is a no-op.

```shell
python change_version.py set-version App.csproj Version 0.2.0
python change_version.py sync-version README.md
```

Without `--version-key` it replaces every occurrence, which is what documentation
files such as `README.md` need. A file that does not mention the version is
reported but does not fail. Pass `--version-key` to limit the replacement to
lines whose key matches instead, in which case a file with no matching line is an
error.

```shell
python change_version.py sync-version manifest.json --version-key version
```

A newline-separated list of files can be handled in a loop:

```shell
while IFS= read -r entry; do
    file="${entry%%|*}"
    key="${entry#*|}"

    if [[ "$entry" == *"|"* ]]; then
        python change_version.py sync-version "$file" --version-key "$key"
    else
        python change_version.py sync-version "$file"
    fi
done <<'EOF'
package.json|version
README.md
EOF
```

### `create-version-file`

`create-version-file` only writes `./.version`. Use it when the new version comes
from somewhere else, such as a git tag. It normalizes the version the same way
`set-version` does, so `v1.2.3` becomes `1.2.3`. It does not write
`./.version_old`, so `sync-version` still needs a `set-version` run first.

```shell
python change_version.py create-version-file "v1.2.3"
```

### Deprecated aliases

The old names still work so existing workflows keep running; Typer additionally marks them as
deprecated in `--help`, and each call prints a warning before forwarding to the new command.

```shell
python change_version.py change-version ...   # -> set-version
python change_version.py replace-version ...  # -> sync-version
```

### GitHub Actions outputs

When `GITHUB_OUTPUT` is set, the commands export these step outputs:

| Command               | Outputs                      |
|-----------------------|------------------------------|
| `set-version`         | `old_version`, `new_version` |
| `create-version-file` | `new_version`                |

## get_version.py

### `get-version`

Fetches a URL and exports the response body as the `get_version` step output. The
script has a single command, so it is called without a subcommand.

```shell
python get_version.py https://example.com/version.txt
```

| Option    | Description                    |
|-----------|--------------------------------|
| `--debug` | Enable `icecream` debug output |
