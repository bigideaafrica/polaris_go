# Polaris Setup

## Overview

Polaris is a modern development workspace manager for distributed compute resources. This repository contains setup scripts to help you install, configure, and manage Polaris on your system.

## System Requirements

- **Operating System**: Linux (supports Ubuntu/Debian-based systems; Windows users need WSL)
- **Python**: Version 3.8 or higher (3.10 recommended)
- **Git**: For cloning repositories
- **Disk Space**: At least 2GB free space recommended

## Installation

1. Clone the repository:
```bash
git clone https://github.com/bigideaafrica/Polaris_go
cd Polaris_go
```

2. Make the script executable:
```bash
chmod +x polaris_manager.sh
```

3. Run the Polaris manager script:
```bash
./polaris_manager.sh
```

4. Follow the interactive prompts to complete installation.

### Windows Users

If you're using Windows, you'll need to install WSL (Windows Subsystem for Linux):

1. The script will help you set up WSL if it detects you're on Windows
2. After WSL is installed, you'll need to copy the script to your WSL environment:
```bash
cp /mnt/c/Users/YourUsername/path/to/polaris_manager.sh ~/
chmod +x ~/polaris_manager.sh
./polaris_manager.sh
```

## Usage

### Managing Polaris

The `polaris_manager.sh` script provides an interactive menu with the following options:

1. **Install Polaris** - First-time installation
2. **Reinstall Polaris** - Reinstall with fresh configuration
3. **Enter Polaris Environment** - Access the Polaris command interface
4. **Uninstall Polaris** - Remove all Polaris components
5. **Check System Status** - Verify system compatibility
6. **System Information** - Display detailed system information
7. **Backup Polaris Configuration** - Save your current settings

### Polaris Commands

Once in the Polaris environment, you can use the following commands:

- `polaris start` - Start Polaris services
- `polaris stop` - Stop Polaris services
- `polaris status` - Check service status
- `polaris logs` - View service logs
- `polaris register` - Register as a new miner
- `polaris --help` - Show all available commands

## Troubleshooting

### Known Issues & Workarounds

- If you have an existing wallet, you may need to regenerate it before starting Polaris
- If services fail to start, try the reinstall option which performs a clean installation
- Python version conflicts can be resolved by installing Python 3.10 through the script
- For WSL users, make sure you're running the script within the Linux environment

### Getting Help

If you encounter problems not covered in this README:

1. Check the logs in the Polaris environment using `polaris logs`
2. Use the troubleshooting option in the manager script
3. Visit the [Polaris documentation](https://github.com/bigideaafrica/polariscloud.git) for detailed information

## License

This project is licensed under the terms of the MIT license.
