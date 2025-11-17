# CLAUDE.md - AI Assistant Guide for nixos-images

## Project Overview

**nixos-images** is a NixOS community project that builds and maintains automatically updated bootable images for NixOS installations. The project extends images created by hydra.nixos.org with weekly automated builds.

- **Repository**: nix-community/nixos-images
- **License**: MIT
- **Primary Language**: Nix
- **Build System**: Nix Flakes
- **Platforms**: x86_64-linux, aarch64-linux

### Purpose

This project creates optimized installation images for:
1. **Remote/unattended installations** (via nixos-anywhere, clan)
2. **Kexec-based installations** from existing Linux systems
3. **USB-bootable ISO installers** with remote access features
4. **Network boot (netboot/iPXE)** installations

## Repository Structure

```
.
├── flake.nix                    # Main Flake configuration defining all packages and modules
├── flake.lock                   # Locked dependency versions
├── build-images.sh              # Script to build and upload images to GitHub releases
├── README.md                    # User-facing documentation
├── LICENSE                      # MIT license
├── .mergify.yml                 # Mergify configuration for PR automation
├── .github/
│   └── workflows/
│       ├── build.yml            # CI workflow for building images on push to main
│       └── update-flake-lock.yml # Automated flake.lock updates twice weekly
└── nix/                         # NixOS modules and configurations
    ├── installer.nix            # Base installer configuration (ZFS, bcachefs, disko, etc.)
    ├── noninteractive.nix       # Optimizations for non-interactive deployments
    ├── networkd.nix             # systemd-networkd configuration
    ├── serial.nix               # Serial console configuration
    ├── restore-remote-access.nix # SSH key and network restoration
    ├── nix-settings.nix         # Nix daemon settings
    ├── latest-zfs-kernel.nix    # ZFS kernel configuration
    ├── zfs-minimal.nix          # Minimal ZFS support
    ├── python-minimal.nix       # Minimal Python environment
    ├── log-network-status.nix   # Network logging utilities
    ├── noveau-workaround.nix    # Nvidia nouveau workarounds
    ├── kexec-installer/
    │   ├── module.nix           # Kexec installer module
    │   ├── kexec-run.sh         # Shell script for kexec execution
    │   ├── restore_routes.py    # Python script for network restoration
    │   ├── test.nix             # VM tests for kexec installer
    │   └── ssh-keys/            # SSH key management
    ├── image-installer/
    │   ├── module.nix           # ISO installer module
    │   ├── hidden-ssh-announcement.nix # Tor hidden service setup
    │   ├── wifi.nix             # WiFi configuration with IWD
    │   └── tests.nix            # Boot tests for ISO installer
    └── netboot-installer/
        └── module.nix           # Netboot installer module
```

## Key Components

### 1. Kexec Installer (`nix/kexec-installer/`)

**Purpose**: Boot NixOS installer from a running Linux system without reboot.

**Key Features**:
- Reuses SSH host keys to preserve `.ssh/known_hosts`
- Restores static IP addresses and routes after kexec
- Reads authorized SSH keys from multiple locations
- 6-second delay before kexec for clean script disconnection
- Minimal variant (`noninteractive`) optimized for automated deployment

**Outputs**:
- `kexec-installer-nixos-unstable` - Standard kexec tarball
- `kexec-installer-nixos-stable` - Stable channel kexec tarball
- `kexec-installer-nixos-{unstable,stable}-noninteractive` - Minimal variants

**Usage Pattern**:
```bash
curl -L <url>/nixos-kexec-installer-noninteractive-x86_64-linux.tar.gz | tar -xzf- -C /root
/root/kexec/run
```

### 2. ISO Image Installer (`nix/image-installer/`)

**Purpose**: USB-bootable installation media optimized for remote installation.

**Key Features**:
- OpenSSH enabled by default
- Random root password generated on each boot
- Tor hidden SSH service for remote access
- QR code with connection info (IP, password, .onion address)
- IWD (iNet Wireless Daemon) for easy WiFi setup
- Network status display on console
- Custom Tango color theme
- Auto-login to root

**Outputs**:
- `image-installer-nixos-unstable` - Latest unstable ISO
- `image-installer-nixos-stable` - Stable release ISO

### 3. Netboot Installer (`nix/netboot-installer/`)

**Purpose**: Network boot via iPXE for diskless installations.

**Key Features**:
- Minimal netboot image with kernel and initrd
- iPXE script for boot configuration
- Dynamic hostname from kernel cmdline
- Full network configuration (DHCP, IPv6 RA, mDNS, LLDP)

**Outputs**:
- `netboot-nixos-unstable` - Kernel, initrd, iPXE script
- `netboot-nixos-stable` - Stable netboot images

### 4. Shared Modules (`nix/`)

#### `installer.nix`
Base configuration for all installers:
- Latest ZFS kernel support (`zfsUnstable`)
- bcachefs filesystem support
- Essential tools: nixos-install-tools, disko, jq, rsync, nixos-facter
- zswap enabled for low-memory systems
- No nixpkgs channel (saves space)
- Emergency initrd access for debugging

#### `noninteractive.nix`
Optimizations for automated deployments:
- Removes interactive tools (nano, nixos-option, strace)
- Disables unnecessary services (sudo, polkit, X11 forwarding)
- Minimal filesystem support (no CIFS, JFS, ReiserFS)
- Uses systemd-sysusers instead of userborn
- dbus-broker instead of dbus for no X11 dependencies
- Significantly reduced closure size

#### `networkd.nix`
Systemd-networkd configuration for consistent networking.

#### `restore-remote-access.nix`
Preserves SSH access and network configuration across kexec.

## Development Workflow

### Prerequisites

```bash
# Install Nix with flakes enabled
nix --version  # Ensure Nix is installed

# Enable flakes (if not already enabled)
mkdir -p ~/.config/nix
echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf
```

### Building Images Locally

```bash
# Build kexec installer (unstable, x86_64)
nix build .#packages.x86_64-linux.kexec-installer-nixos-unstable

# Build kexec installer (stable, aarch64)
nix build .#packages.aarch64-linux.kexec-installer-nixos-stable

# Build ISO installer (unstable)
nix build .#packages.x86_64-linux.image-installer-nixos-unstable

# Build netboot image (stable)
nix build .#packages.x86_64-linux.netboot-nixos-stable

# Build all images for current system
nix build .#packages.$(nix eval --raw --impure --expr builtins.currentSystem)
```

### Running Tests

```bash
# Run kexec installer test (x86_64 only)
nix build .#checks.x86_64-linux.kexec-installer-unstable

# Run shellcheck on kexec-run.sh
nix build .#checks.x86_64-linux.shellcheck

# Run ISO boot tests
nix build .#checks.x86_64-linux.boot-test-nixos-unstable

# Run all checks
nix flake check
```

### Testing Images in VMs

```bash
# Test kexec installer
result=$(nix build --print-out-paths .#packages.x86_64-linux.kexec-installer-nixos-unstable-noninteractive)
# Extract and test in VM (see nix/kexec-installer/test.nix for example)

# Test ISO image
result=$(nix build --print-out-paths .#packages.x86_64-linux.image-installer-nixos-unstable)
# Boot ISO in VM or write to USB
```

### Updating Dependencies

```bash
# Update flake.lock manually
nix flake update

# Or update specific inputs
nix flake lock --update-input nixos-unstable
nix flake lock --update-input nixos-stable
```

**Automated Updates**: GitHub Actions runs `update-flake-lock` workflow twice weekly (Monday & Thursday) and automatically creates PRs.

## Build System Details

### Flake Structure

The `flake.nix` defines:

1. **Inputs**:
   - `nixos-unstable`: Latest nixpkgs-unstable
   - `nixos-stable`: Current stable release (25.05)
   - Both use shallow git clones for faster fetches

2. **Outputs**:
   - `packages.<system>.*`: All buildable images
   - `nixosModules.*`: Reusable NixOS modules
   - `checks.<system>.*`: Tests and validation

3. **Binary Cache**:
   - Uses nix-community.cachix.org for cached builds
   - Configured in `nixConfig` section

### Build Script (`build-images.sh`)

Shell script for building and uploading images:

**Functions**:
- `build_netboot_image()`: Builds kernel, initrd, iPXE script
- `build_kexec_installer()`: Builds kexec tarball (standard and noninteractive)
- `build_image_installer()`: Builds ISO image

**Usage**:
```bash
./build-images.sh [TAG] [ARCH]
# TAG: nixos-unstable (default) or nixos-24.11
# ARCH: x86_64-linux (default) or aarch64-linux

# Examples:
./build-images.sh nixos-unstable x86_64-linux
./build-images.sh nixos-24.11 aarch64-linux
```

**Release Upload**:
- Creates GitHub release if not exists
- Uploads all built artifacts with `gh release upload --clobber`

## CI/CD Pipeline

### Build Workflow (`.github/workflows/build.yml`)

**Triggers**: Push to `main`, manual dispatch, repository dispatch

**Matrix Strategy**:
- Tags: `nixos-24.11`, `nixos-unstable`
- Platforms: `ubuntu-latest` (x86_64), `nscloud-ubuntu-22.04-arm64-4x16` (aarch64)

**Steps**:
1. Checkout repository
2. Install Nix via `cachix/install-nix-action@v31`
3. Run `build-images.sh` with matrix parameters
4. Upload artifacts to GitHub releases

### Update Flake Lock Workflow (`.github/workflows/update-flake-lock.yml`)

**Triggers**: Manual dispatch, scheduled (Monday & Thursday 00:00 UTC)

**Steps**:
1. Checkout repository
2. Install Nix
3. Run `DeterminateSystems/update-flake-lock@v24`
4. Create PR with label `merge-queue` for automated merging

### Mergify Configuration (`.mergify.yml`)

**Auto-merge Rules**:
- PRs labeled `merge-queue` or `dependencies`
- Must pass `buildbot/nix-build` check
- Uses rebase merge method

## Testing Strategy

### Unit Tests

1. **Shellcheck**: Validates `kexec-run.sh` syntax
2. **Kexec Installer Test** (`nix/kexec-installer/test.nix`):
   - VM-based test simulating kexec process
   - Tests network restoration and SSH access

### Integration Tests

**ISO Boot Tests** (`nix/image-installer/tests.nix`):
- BIOS boot test
- UEFI boot test
- Tests for both unstable and stable channels

### Manual Testing Checklist

When modifying installers, test:

**Kexec Installer**:
- [ ] Tarball extracts correctly
- [ ] Kexec executes without errors
- [ ] SSH host keys preserved
- [ ] Static network config restored
- [ ] Authorized keys accessible
- [ ] Can run `nixos-anywhere` successfully

**ISO Installer**:
- [ ] Boots on BIOS systems
- [ ] Boots on UEFI systems
- [ ] Root password displayed correctly
- [ ] QR code renders properly
- [ ] SSH accessible (local network)
- [ ] Tor hidden service reachable
- [ ] WiFi connects via `iwctl`
- [ ] Network status updates correctly

**Netboot Installer**:
- [ ] iPXE script loads kernel and initrd
- [ ] Boots in network environment
- [ ] SSH accessible after boot
- [ ] Network autoconfiguration works

## Key Conventions and Best Practices

### Nix Code Style

1. **Module Imports**: Use explicit paths relative to module location
   ```nix
   imports = [
     (modulesPath + "/installer/netboot/netboot-minimal.nix")
     ../installer.nix
   ];
   ```

2. **Options**: Define module-specific options with clear descriptions
   ```nix
   options = {
     system.kexec-installer.name = lib.mkOption {
       type = lib.types.str;
       default = "nixos-kexec-installer";
       description = "The variant of the kexec installer to use.";
     };
   };
   ```

3. **State Version**: Always set to current release for installers
   ```nix
   system.stateVersion = config.system.nixos.release;
   ```

4. **Conditionals**: Use `lib.mkForce` to override upstream defaults
   ```nix
   services.getty.autologinUser = lib.mkForce "root";
   ```

5. **Version Compatibility**: Handle nixpkgs version differences
   ```nix
   // (if lib.versionAtLeast lib.version "25.03pre" then {
     image.baseName = lib.mkForce "nixos-installer-${pkgs.system}";
   } else {
     isoImage.isoName = lib.mkForce "nixos-installer-${pkgs.system}.iso";
   })
   ```

### Shell Script Conventions

1. **Shebang**: Use nix-shell with required packages
   ```bash
   #!/usr/bin/env nix-shell
   #!nix-shell -p nix -p coreutils -p bash -p gh -i bash
   ```

2. **Safety**: Enable strict error handling
   ```bash
   set -xeuo pipefail
   shopt -s lastpipe
   ```

3. **Shellcheck**: All scripts must pass shellcheck validation
   ```nix
   ${pkgs.shellcheck}/bin/shellcheck $out
   ```

### File Naming

- **Nix modules**: `kebab-case.nix` (e.g., `restore-remote-access.nix`)
- **Shell scripts**: `kebab-case.sh` (e.g., `build-images.sh`)
- **Python scripts**: `snake_case.py` (e.g., `restore_routes.py`)
- **Output artifacts**: Consistent format
  - Kexec: `nixos-kexec-installer[-noninteractive]-${arch}.tar.gz`
  - ISO: `nixos-installer-${arch}.iso`
  - Netboot: `bzImage-${arch}`, `initrd-${arch}`, `netboot-${arch}.ipxe`

### Commit Messages

Follow conventional commits:
- `feat:` - New features
- `fix:` - Bug fixes
- `refactor:` - Code restructuring
- `chore:` - Maintenance tasks
- `ci:` - CI/CD changes
- `docs:` - Documentation updates

Examples:
```
feat: Update flake.nix stable to 25.05
fix: Restore SSH keys in kexec installer
refactor: Split noninteractive module
chore: Update flake.lock
```

### PR Guidelines

1. **Branch Naming**: Use descriptive names
   - `feature/add-wifi-support`
   - `fix/kexec-network-restore`
   - `chore/update-dependencies`

2. **Labels**: Add appropriate labels
   - `merge-queue`: For automated merging via Mergify
   - `dependencies`: For dependency updates

3. **Testing**: Ensure all checks pass before merging
   - `nix flake check` succeeds
   - CI builds complete for all platforms

4. **Size Optimization**: For noninteractive images, verify closure size
   ```bash
   # Check closure size
   nix path-info -rsSh ./result
   ```

## Common Tasks for AI Assistants

### Adding a New Feature to Installers

1. **Create/modify module** in `nix/`
2. **Import module** in appropriate installer module
3. **Test locally** with `nix build`
4. **Add tests** if applicable (in `test.nix` or `tests.nix`)
5. **Update documentation** (README.md and this file)
6. **Run checks**: `nix flake check`

### Updating to New NixOS Release

1. **Update flake.nix**: Change `nixos-stable.url` reference
   ```nix
   inputs.nixos-stable.url = "git+https://github.com/NixOS/nixpkgs?shallow=1&ref=nixos-25.11";
   ```
2. **Update flake.lock**: `nix flake update nixos-stable`
3. **Update build workflow**: Add new tag to matrix in `.github/workflows/build.yml`
4. **Update README.md**: Document new version availability
5. **Test all packages**: Build and test on both architectures
6. **Update deprecation notices**: Remove old versions when appropriate

### Debugging Build Failures

1. **Check logs** in GitHub Actions for specific errors
2. **Reproduce locally**:
   ```bash
   nix build --show-trace .#packages.x86_64-linux.PACKAGE_NAME
   ```
3. **Test in clean environment**:
   ```bash
   nix build --option pure-eval true .#PACKAGE_NAME
   ```
4. **Check dependencies**:
   ```bash
   nix path-info -r ./result
   ```
5. **Verify shellcheck**:
   ```bash
   nix build .#checks.x86_64-linux.shellcheck
   ```

### Reducing Image Size

For noninteractive images:
1. **Disable services**: Mark unnecessary services with `enable = false`
2. **Remove packages**: Use `lib.mkForce []` for package lists
3. **Disable documentation**: `documentation.enable = false`
4. **Use minimal alternatives**: dbus-broker, python3Minimal, pkgsStatic
5. **Verify reduction**:
   ```bash
   nix path-info -rsSh ./result | tail -1
   ```

### Adding New Package Output

1. **Define function** in flake.nix `packages` section:
   ```nix
   packages = forAllSystems (system: {
     new-installer = (nixpkgs.legacyPackages.${system}.nixos [
       self.nixosModules.new-installer
     ]).config.system.build.output;
   });
   ```
2. **Create module** in `nix/new-installer/module.nix`
3. **Add to nixosModules**: `nixosModules.new-installer = ./nix/new-installer/module.nix`
4. **Update build-images.sh**: Add build function and call in `main()`
5. **Test build**: `nix build .#packages.x86_64-linux.new-installer`
6. **Add to CI**: Update matrix or add new step if needed

## Important Notes for AI Assistants

### When Modifying Code

1. **Always preserve existing functionality** unless explicitly asked to change
2. **Test on both architectures** (x86_64 and aarch64) when possible
3. **Maintain backward compatibility** with existing users' workflows
4. **Consider closure size impact** for noninteractive images
5. **Update all relevant documentation** (CLAUDE.md, README.md, comments)
6. **Follow Nix best practices**: Use lib functions, avoid IFD (Import From Derivation)

### Common Pitfalls

1. **Don't use `nix-env`**: All packages should be in flake outputs
2. **Avoid fetchurl in modules**: Pre-fetch and include in repository if needed
3. **Don't hardcode paths**: Use `${pkgs.package}/bin/binary` syntax
4. **Check NixOS version compatibility**: Use version conditionals when needed
5. **Don't break shellcheck**: All shell scripts must pass validation
6. **Respect noninteractive optimization**: Don't add heavyweight deps to minimal images

### Security Considerations

1. **SSH security**: Never commit private keys or credentials
2. **Password generation**: Use cryptographically secure random generation
3. **Network security**: Default to secure protocols (HTTPS, SSH)
4. **Minimal attack surface**: Remove unnecessary services in installers
5. **Tor hidden service**: Properly generate and protect .onion addresses

### Performance Optimization

1. **Compression**: Use appropriate compression (zstd for ISO, xz for initrd)
2. **Parallel builds**: Leverage Nix's parallel build capability
3. **Binary cache**: Ensure cachix.org cache is configured
4. **Shallow git clones**: Use `shallow=1` for flake inputs
5. **zswap configuration**: Optimize for low-memory systems

## Resources and References

### Documentation

- [NixOS Manual](https://nixos.org/manual/nixos/stable/)
- [Nixpkgs Manual](https://nixos.org/manual/nixpkgs/stable/)
- [Nix Flakes](https://nixos.wiki/wiki/Flakes)
- [nixos-anywhere Documentation](https://github.com/numtide/nixos-anywhere)
- [disko Documentation](https://github.com/nix-community/disko)

### Related Projects

- **nixos-anywhere**: Automated NixOS installation using kexec images
- **clan**: Declarative NixOS deployment framework
- **disko**: Declarative disk partitioning for NixOS
- **nixos-facter**: Modern alternative to nixos-generate-config

### Community

- **GitHub**: https://github.com/nix-community/nixos-images
- **NixOS Discourse**: https://discourse.nixos.org/
- **NixOS Matrix**: #nixos:matrix.org

## Troubleshooting

### Build Errors

**Error: "no space left on device"**
- Clean Nix store: `nix-collect-garbage -d`
- Check disk space: `df -h /nix`

**Error: "cached failure of attribute"**
- Clear evaluation cache: `nix flake check --no-eval-cache`

**Error: "does not depend on flake"**
- Update flake.lock: `nix flake update`

### Testing Issues

**VM test hangs**
- Increase timeout in test configuration
- Check QEMU console output: Add `-serial stdio`

**SSH connection refused in ISO test**
- Verify sshd started: Check systemd status
- Check firewall rules: Ensure port 22 open

### Image Problems

**Kexec fails with OOM**
- Increase RAM allocation (minimum 1GB required)
- Enable zswap (already configured)

**ISO won't boot**
- Verify UEFI/BIOS mode match
- Check secure boot is disabled
- Verify USB writing method (use `dd` or Etcher)

**Netboot times out**
- Check DHCP server configuration
- Verify iPXE script URL accessible
- Ensure sufficient network bandwidth

## Version History

- **2025-11**: Updated to NixOS 25.05 stable
- **2024-11**: NixOS 24.11 release support
- **2024**: Initial project setup and automation

---

**Last Updated**: 2025-11-17

This guide is maintained for AI assistants working on the nixos-images project. For user-facing documentation, see README.md.
