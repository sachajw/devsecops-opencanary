# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenCanary is a multi-protocol network honeypot designed to detect attackers on internal networks. It runs as a daemon implementing multiple network protocols (SSH, HTTP, FTP, MySQL, etc.) and alerts when attackers interact with them.

## Development Commands

### Setup and Installation
```bash
# Create and activate virtual environment
virtualenv env/
. env/bin/activate

# Install from source
python setup.py sdist
pip install dist/opencanary-*.tar.gz

# Install test dependencies
pip install -r opencanary/test/requirements.txt
```

### Configuration
```bash
# Create default config file at /etc/opencanaryd/opencanary.conf
opencanaryd --copyconfig

# For testing, copy the test config
cp opencanary/test/opencanary.conf .
```

### Running
```bash
# Start daemon (production)
opencanaryd --start --uid=nobody --gid=nogroup

# Run in foreground (development)
opencanaryd --dev

# Stop daemon
opencanaryd --stop

# Check version
opencanaryd --version
```

### Testing and Linting
```bash
# Run tests (requires opencanaryd to be running)
pytest -s

# Run pre-commit checks (black, flake8)
pre-commit run --all-files
```

### Docker
```bash
# Build and run with docker-compose
docker compose up latest
docker compose up stable
```

## Architecture

### Core Components

- **`bin/opencanaryd`**: Shell script entry point that wraps twistd
- **`bin/opencanary.tac`**: Twisted application configuration; loads and starts all enabled modules
- **`opencanary/config.py`**: Configuration management (JSON-based, loads from `/etc/opencanaryd/opencanary.conf`, `~/.opencanary.conf`, or `./opencanary.conf`)
- **`opencanary/logger.py`**: Pluggable logging system with multiple output handlers

### Module System

Protocol modules are located in `opencanary/modules/`. Each module:
- Inherits from `CanaryService` (in `opencanary/modules/__init__.py`)
- Has a `NAME` class attribute matching the config section (e.g., `"ssh"` maps to `ssh.enabled`, `ssh.port`)
- Implements either `getService()` (returns Twisted service) or `startYourEngines()` (for non-Twisted setup)
- Uses `self.log(logdata)` to report events, which includes src/dst IP and port info

**Example module pattern** (see `opencanary/modules/example0.py`):
```python
from opencanary.modules import CanaryService
from twisted.internet.protocol import Protocol, Factory

class MyProtocol(Protocol):
    def dataReceived(self, data):
        self.factory.log({"PASSWORD": data.strip()}, transport=self.transport)

class CanaryMyService(Factory, CanaryService):
    NAME = "myservice"
    protocol = MyProtocol

    def __init__(self, config=None, logger=None):
        CanaryService.__init__(self, config, logger)
        self.port = 8000
        self.logtype = logger.LOG_USER_0
```

### Adding New Modules

1. Create module file in `opencanary/modules/`
2. Add to `MODULES` list in `bin/opencanary.tac`, OR
3. Register via setuptools entry point `canary.usermodule` for external packages

### Available Protocol Modules

`ftp`, `git`, `http`, `https`, `httpproxy`, `mysql`, `mssql`, `ntp`, `rdp`, `redis`, `sip`, `smb` (Linux only), `snmp` (requires Scapy), `llmnr` (requires Scapy), `ssh`, `tcpbanner`, `telnet`, `tftp`, `vnc`, `portscan` (Linux only, uses iptables)

### Platform-Specific Notes

- **SNMP/LLMNR**: Requires `pip install scapy pcapy-ng`
- **Samba/SMB**: Requires Samba installation; see wiki for setup
- **Portscan**: Linux only; modifies iptables rules

## Configuration

Configuration is JSON-based. Key patterns:
- `"<module>.enabled": true/false` - Enable/disable a module
- `"<module>.port": <int>` - Port to listen on
- `"logger": {...}` - Logger configuration (class, kwargs)

See `opencanary/test/opencanary.conf` for a complete example with all modules.

## Version

Current version is defined in `opencanary/__init__.py` as `__version__`.
