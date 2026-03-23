# NewExportToHASS

Fork of [ExportToHASS](https://github.com/adizanni/ExportToHASS) by adizanni.

SweetHome3D plugin that exports a 3D home model to Wavefront OBJ format optimized for the [floor3d-card](https://github.com/adizanni/floor3d-card) in Home Assistant.

**Why a fork?** This fork renames the internal Java package and plugin identifier so that `NewExportToHASS` and the original `ExportToHASS` can be installed simultaneously in SweetHome3D without class-loading conflicts.

---

## Plugin Installation

1. Download the latest `NewExportToHASSPlugin-<version>-sh3d6.6.sh3p` from [Releases](../../releases/latest).
2. Copy it to your SweetHome3D plugins folder:
   - **Linux:** `~/.eteks/sweethome3d/plugins/`
   - **Windows/macOS:** double-click the `.sh3p` file.
3. Restart SweetHome3D.
4. The action appears under **Tools → Export obj to HASS**.

The plugin is compiled against SweetHome3D 6.6. It requires SweetHome3D ≥ 6.5 and Java ≥ 1.5.

---

## How to use

Open your model, click **Tools → Export obj to HASS**, choose a destination ZIP file. Once the ZIP is created, unzip it to `/config/www/<your-model-folder>/` on your Home Assistant instance. The model file is named `home.obj` by default. The ZIP also contains a JSON file listing all `object_id` values, which feeds the dropdown in the floor3d-card editor.

### How it works

Objects are named using the **object name** set in SweetHome3D's properties panel — not an auto-generated index — so that IDs stay stable between exports even when other objects are removed.

- Append `#` to an object name to group all its sub-components under one ID (the `#` is stripped on export).
- Multi-level homes: each object is prefixed with `lvl000`, `lvl001`, … matching the level index.
- Naming rules — valid characters in object names:

```
a–z  A–Z  0–9  _ (underscore)
# is allowed as a grouping marker and stripped on export
```

### Object type mapping

| SweetHome3D type   | OBJ object name pattern            |
|--------------------|------------------------------------|
| Furniture piece    | `[lvlNNN]<object_name>`            |
| Furniture group    | `[lvlNNN]<group_name>_<item_name>` |
| Wall               | `[lvlNNN]wall_<index>`             |
| Room               | `[lvlNNN]room_<index>`             |
| Other              | `[lvlNNN]object_<index>`           |

---

## Building from source

Requires JDK 7+ and [Apache Ant](https://ant.apache.org/).

```bash
ant          # clean → compile → package
# Output: dist/NewExportToHASSPlugin-<version>-sh3d6.6.sh3p
```

Releases are also built automatically by the GitHub Actions workflow (`.github/workflows/release.yml`) when a version tag `v*` is pushed.

---

## Architecture

See [docs/architecture.md](docs/architecture.md) for a full technical description with diagrams.

---

## License

GNU General Public License v3. See [LICENSE](LICENSE).

Original work © 2008 Emmanuel PUYBARET / eTeks. Fork maintained separately.
