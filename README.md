# minecraft-modded-game-template

A template repository for creating a modded Minecraft game experience using the
[minecraft-server-snap](https://github.com/kenvandine/minecraft-server-snap) framework.

Push a tag → get a GitHub release with a server artifact, a Linux launcher, and a
Windows launcher — automatically.

---

## Using this template

Click **"Use this template"** → **"Create a new repository"** on GitHub, then follow
the steps below.

---

## Setup (5 minutes)

### Step 1 — Edit `pack.yaml`

Open `pack.yaml` and customize it for your game:

```yaml
name: "My Awesome Modpack"     # ← change this
version: "1.0.0"
minecraft_version: "1.21.1"
mod_loader: fabric
mod_loader_version: "latest"
installer_version: "latest"

mods:
  - name: "Fabric API"
    url: "https://cdn.modrinth.com/data/P7dR8mSH/versions/adK8OREi/fabric-api-0.105.0+1.21.1.jar"
    side: both
  # Add your mods here
```

See the [pack YAML reference](https://github.com/kenvandine/minecraft-server-snap/blob/main/docs/pack-yaml-reference.md)
for all available fields.

### Step 2 — (Optional) Set up Microsoft auth

For players to sign in with their Microsoft account for online play, you need a free
Azure app registration. Follow the
[Azure setup guide](https://github.com/kenvandine/minecraft-server-snap/blob/main/docs/azure-setup.md)
(takes about 5 minutes), then add your client ID to `pack.yaml`:

```yaml
azure_client_id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

If you skip this step, the launcher will work for LAN play and offline-mode servers.

### Step 3 — Publish your first release

```bash
git add pack.yaml
git commit -m "Initial pack configuration"
git tag v1.0.0
git push origin main --tags
```

GitHub Actions will automatically:
1. Download the Fabric server JAR and your mods
2. Build `server.tar.xz`
3. Build a Linux AppImage launcher
4. Build a Windows exe installer
5. Create a GitHub release with all three artifacts

Watch the progress under **Actions** in your repository.

---

## Adding and updating mods

Find the mod on [modrinth.com](https://modrinth.com), navigate to **Versions**, select
the version matching your `minecraft_version` and `fabric` loader, right-click the
download button and copy the URL.

Add it to `pack.yaml`:

```yaml
mods:
  - name: "Iris Shaders"
    url: "https://cdn.modrinth.com/data/YL57xq9U/versions/.../iris-1.8.0+mc1.21.1.jar"
    side: client
```

Then publish a new release:

```bash
# Edit pack.yaml — bump version, add/update mods
git add pack.yaml
git commit -m "Add Iris Shaders"
git tag v1.1.0
git push origin main --tags
```

---

## Installing on a server

Install the [minecraft-server snap](https://github.com/kenvandine/minecraft-server-snap):

```bash
sudo snap install minecraft-server
```

Then install your modpack from the release URL:

```bash
sudo minecraft-server.install-pack \
  https://github.com/YOUR-ORG/YOUR-REPO/releases/download/v1.0.0/server.tar.xz

sudo snap restart minecraft-server.server
sudo snap logs -f minecraft-server.server
```

---

## Playing

Direct players to your GitHub releases page. They download:

- **Linux**: `Your-Modpack-1.0.0.AppImage` → `chmod +x *.AppImage && ./*.AppImage`
- **Windows**: `Your-Modpack-Setup-1.0.0.exe` → run the installer

On first launch, the launcher automatically downloads Java 21, Minecraft, Fabric, and
installs the bundled mods (~500 MB, one time). After that, it's just click Play.

---

## Local testing

Install `game-create` to test your pack locally before pushing a tag:

```bash
# Linux
curl -L -o /usr/local/bin/game-create \
  https://github.com/kenvandine/minecraft-server-snap/releases/latest/download/game-create-linux
chmod +x /usr/local/bin/game-create

# Build artifacts locally
game-create build pack.yaml --output ./dist

# Test the server artifact
sudo minecraft-server.install-pack dist/server.tar.xz
```

To test the launcher locally, see the
[launcher build guide](https://github.com/kenvandine/minecraft-server-snap/blob/main/docs/launcher.md#building-locally).

---

## Repository structure

```
your-game/
├── pack.yaml                    Your modpack definition — edit this
├── .github/
│   └── workflows/
│       └── release.yml          Triggers on version tags → builds + publishes release
└── .gitignore
```

That's it. The build tools, launcher template, and CI logic all live in
[minecraft-server-snap](https://github.com/kenvandine/minecraft-server-snap).

---

## Workflow reference

The workflow in `.github/workflows/release.yml` calls the reusable workflow from
`minecraft-server-snap`. You generally don't need to edit it, but you can customize:

```yaml
jobs:
  release:
    uses: kenvandine/minecraft-server-snap/.github/workflows/reusable-pack-release.yml@main
    with:
      config: pack.yaml               # path to your YAML
      tag: ${{ github.ref_name }}     # release tag
      tools-version: "v0.2.0"        # pin game-create version (optional)
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

See [github-workflows.md](https://github.com/kenvandine/minecraft-server-snap/blob/main/docs/github-workflows.md)
for full documentation.

---

## Troubleshooting

**Workflow fails at "Build artifacts" — mod download error**
Check that all mod URLs in `pack.yaml` are valid direct download links to `.jar` files.
Open the URL in a browser — it should immediately download a JAR.

**Workflow fails at "Build launcher" — electron-builder error**
This is usually a Node.js version issue. The workflow uses Node 20 — ensure you haven't
accidentally pinned an older version in a workflow override.

**Players get "Failed to get Minecraft profile"**
The player authenticated with Microsoft but does not own Minecraft Java Edition.
The game requires a purchased copy.

**Players see "No Azure client ID configured"**
Add `azure_client_id` to `pack.yaml`. See the
[Azure setup guide](https://github.com/kenvandine/minecraft-server-snap/blob/main/docs/azure-setup.md).

**Server won't start after install-pack**
Run `sudo snap logs minecraft-server.server`. Common causes:
- Port 25565 already in use (another server running)
- Insufficient memory (`java_args` in pack.yaml requests more RAM than available)
