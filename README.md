# Tukkercraft 3 - Minecraft Modpack Workflow

This repository contains a complete workflow for deploying Minecraft modpacks using GitHub Actions and Docker Compose.

## Overview

The workflow supports:
- Extracting and processing `.mrpack` files (Modrinth modpack format)
- Downloading mod JARs from Modrinth CDN
- Setting up a Minecraft server with itzg/minecraft-server
- Serving a companion webpage with nginx

## Structure

```
.
├── .github/
│   └── workflows/
│       ├── deploy-modpack.yml   # Full deployment: extracts, downloads mods, sets up server
│       └── extract-modpack.yml   # Simple extraction only
├── docker-compose.yaml           # Docker Compose configuration
├── *.mrpack                      # Modrinth modpack file(s)
├── data/                         # Minecraft server data (created by workflow)
│   ├── mods/                     # Mod JARs
│   └── config/                   # Configuration files
├── webpage/                      # Webpage files (created by workflow)
│   └── index.html                # Default webpage
├── nginx.conf                    # Nginx configuration (created by workflow)
└── .env                          # Environment variables (created by workflow)
```

## Quick Start

### 1. Add your .mrpack file

Place your Modrinth modpack file (`.mrpack`) in the root of the repository and commit it.

### 2. Trigger the workflow

The workflow can be triggered in two ways:

#### Automatic (on push)
The `deploy-modpack.yml` workflow runs automatically when you push a `.mrpack` file to the repository.

#### Manual
1. Go to **Actions** tab in your GitHub repository
2. Select **Deploy Modpack** workflow
3. Click **Run workflow**
4. Optionally specify the `.mrpack` file name (defaults to first one found)

### 3. Run the server

After the workflow completes:

```bash
# Start the services
docker-compose up -d

# View logs
docker-compose logs -f minecraft

# Stop the services
docker-compose down
```

## Workflows

### deploy-modpack.yml

This is the main workflow that:
1. Extracts the `.mrpack` file (which is a ZIP archive)
2. Parses the `modrinth.index.json` to get mod information
3. Downloads all mod JARs from Modrinth CDN
4. Prepares the server files in the `data/` directory
5. Sets up the webpage in the `webpage/` directory
6. Creates nginx configuration
7. Generates a `.env` file with appropriate settings
8. Commits all changes back to the repository

**Triggers:**
- Push to repository when `.mrpack` files change
- Manual trigger via GitHub Actions UI

**Inputs:**
- `mrpack_file`: Name of the `.mrpack` file to deploy (optional)

### extract-modpack.yml

This is a simpler workflow that only extracts the `.mrpack` file and displays its contents without downloading mods.

**Triggers:**
- Manual trigger via GitHub Actions UI

**Inputs:**
- `mrpack_file`: Name of the `.mrpack` file to extract (optional)

## Docker Compose Configuration

The `docker-compose.yaml` file defines two services:

### minecraft
- **Image**: `itzg/minecraft-server:latest`
- **Ports**: 
  - `${SERVER_PORT}:25565` - Minecraft server port
  - `${RCON_PORT}:25575` - RCON port
- **Volumes**: `./data:/data` - Persistent server data
- **Environment Variables**:
  - `EULA`: Must be "TRUE" to accept Minecraft EULA
  - `VERSION`: Minecraft version (extracted from modpack)
  - `TYPE`: Mod loader type (neoforge, forge, fabric, etc.)
  - `MEMORY`: JVM memory allocation
  - And many more server settings

### webpage
- **Image**: `nginx:alpine`
- **Ports**: `${WEB_PORT}:80` - Webpage port
- **Volumes**:
  - `./webpage:/usr/share/nginx/html:ro` - Webpage files
  - `./nginx.conf:/etc/nginx/conf.d/default.conf:ro` - Nginx config

## Environment Variables

The workflow generates a `.env` file with the following variables:

```bash
# Minecraft Server Configuration
MC_VERSION=<minecraft_version_from_modpack>
MODLOADER=neoforge
MEMORY=8G
SERVER_PORT=25565
RCON_PORT=25575
ENABLE_RCON=true
RCON_PASSWORD=<random_hex_string>
DIFFICULTY=normal
MODE=survival
MOTD=<modpack_name> - <modpack_version>
ONLINE_MODE=true

# Webpage Configuration
WEB_PORT=8080
```

You can override any of these by creating your own `.env` file before running the workflow.

## Customization

### Webpage

The workflow will copy files from the `src/` directory to `webpage/` if they exist. You can add your own:
- `index.html` - Main webpage
- `app.py` - Python application (if using a different web server)
- `static/` - Static files (CSS, JS, images)

If no `index.html` exists, a default one will be created with modpack information.

### Nginx Configuration

The workflow creates a basic `nginx.conf`. You can customize it by:
1. Creating your own `nginx.conf` before running the workflow
2. Or editing it after the workflow runs

### Server Configuration

All Minecraft server settings can be configured via environment variables in the `.env` file. See the [itzg/minecraft-server](https://github.com/itzg/docker-minecraft-server) documentation for all available options.

## How It Works

### .mrpack File Structure

The `.mrpack` file is a ZIP archive containing:
- `modrinth.index.json` - Modpack manifest with mod definitions
- `overrides/` - Directory with:
  - `config/` - Configuration files
  - `mods/` - Some mod JARs (optional)

### Mod Download Process

1. The workflow extracts the `.mrpack` file
2. It reads `modrinth.index.json` to get the list of mods
3. For each mod, it:
   - Checks if the mod already exists in `overrides/mods/`
   - If not, downloads it from the Modrinth CDN URL specified in the index
4. All mods are placed in `data/mods/` for the server

### File Structure After Deployment

```
data/
├── mods/              # All mod JARs
│   ├── mod1.jar
│   ├── mod2.jar
│   └── ...
└── config/            # Configuration files from overrides
    ├── mod1-config.toml
    ├── mod2-config.json
    └── ...

webpage/
├── index.html         # Webpage
├── app.py            # Optional Python app
└── static/           # Optional static files

nginx.conf            # Nginx configuration
docker-compose.yaml   # Docker Compose config
.env                  # Environment variables
```

## Troubleshooting

### Workflow fails with "No .mrpack file found"
- Make sure your `.mrpack` file is in the root of the repository
- Check that the filename ends with `.mrpack`

### Mod downloads fail
- The Modrinth CDN might be rate-limiting. Try running the workflow again later.
- Some mods might have been removed from Modrinth.

### Server fails to start
- Check the logs: `docker-compose logs minecraft`
- Make sure the `MC_VERSION` and `TYPE` (modloader) match your modpack
- Verify that all required mods are in the `data/mods/` directory

### Webpage not accessible
- Check that the nginx container is running: `docker-compose ps`
- Verify the port mapping in `docker-compose.yaml`
- Check the nginx logs: `docker-compose logs webpage`

## References

- [Modrinth](https://modrinth.com/) - Mod hosting platform
- [mrpack format](https://github.com/pmmp/ModrinthDL) - Modrinth pack format
- [itzg/minecraft-server](https://github.com/itzg/docker-minecraft-server) - Docker image for Minecraft server
- [Fabulously-Optimized/mrpack-to-zip](https://github.com/Fabulously-Optimized/mrpack-to-zip) - Inspiration for mod downloading

## License

This workflow is provided as-is. The modpacks and mods are subject to their respective licenses.
