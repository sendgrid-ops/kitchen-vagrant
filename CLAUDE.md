# CLAUDE.md - kitchen-vagrant Documentation

## Repository Overview

**Repository:** kitchen-vagrant
**Purpose:** Vagrant driver for Test Kitchen - enables automated testing of infrastructure code using Vagrant-managed virtual machines
**Language:** Ruby
**Current Version:** 0.15.0
**License:** Apache 2.0

### What This Repository Does

This is a Test Kitchen driver that integrates Vagrant with Test Kitchen. It allows developers to:
- Automatically provision VMs using Vagrant for testing cookbooks, roles, and infrastructure code
- Generate Vagrantfiles dynamically based on test configurations
- Support multiple Vagrant providers (VirtualBox, VMware Fusion/Workstation, cloud providers)
- Manage VM lifecycle (create, converge, verify, destroy) through Test Kitchen's unified interface

### Key Concepts

**Test Kitchen:** A framework for testing infrastructure code in isolation. It handles the orchestration of creating VMs, provisioning them, running tests, and tearing them down.

**Vagrant:** A tool for building and managing virtual machine environments. This driver uses Vagrant CLI commands to manage VMs.

**Driver Pattern:** Test Kitchen uses a driver abstraction to support different VM platforms. This is the Vagrant implementation of that abstraction.

**Sandboxed Environments:** Each test instance gets its own isolated directory with a unique Vagrantfile, preventing conflicts between parallel test runs.

## Architecture

### Component Structure

```
kitchen-vagrant/
├── lib/
│   └── kitchen/
│       └── driver/
│           ├── vagrant.rb           # Main driver implementation
│           └── vagrant_version.rb   # Version constant
├── templates/
│   └── Vagrantfile.erb             # ERB template for Vagrantfiles
├── kitchen-vagrant.gemspec         # Gem specification
└── Rakefile                        # Build tasks
```

### How It Works

1. **Configuration:** User configures Test Kitchen with `.kitchen.yml`:
   ```yaml
   driver:
     name: vagrant
     provider: virtualbox
     customize:
       memory: 1024
   platforms:
     - name: ubuntu-14.04
   ```

2. **Vagrantfile Generation:** Driver renders `Vagrantfile.erb` template with config values, creating a unique Vagrantfile in `.kitchen/kitchen-vagrant/<instance-name>/`

3. **VM Lifecycle:**
   - **Create:** Runs `vagrant up --no-provision` to boot VM
   - **Converge:** Test Kitchen SSHs in and runs provisioner (Chef, Puppet, etc.)
   - **Verify:** Test Kitchen runs tests inside VM
   - **Destroy:** Runs `vagrant destroy -f` and removes sandbox directory

4. **SSH Connection:** Driver extracts SSH details from `vagrant ssh-config` and passes them to Test Kitchen's SSH transport

### Key Design Decisions

**CLI-Based Integration:** Uses Vagrant CLI commands instead of Vagrant's Ruby API. This approach:
- Doesn't require Vagrant gems/plugins
- Works with any Vagrant version
- Avoids Ruby version compatibility issues
- Simple and maintainable

**Isolated Sandboxes:** Each instance gets its own directory to prevent:
- File locking conflicts during parallel runs
- State leakage between test instances
- Accidental cross-contamination

**No Parallel Create/Destroy:** Vagrant uses file locks that conflict during parallel operations, so these actions are serialized (see `no_parallel_for :create, :destroy`)

## File Structure and Organization

### Core Ruby Files

**lib/kitchen/driver/vagrant.rb** (Main driver, ~236 lines)
- `Kitchen::Driver::Vagrant` class inherits from `Kitchen::Driver::SSHBase`
- Configuration management with `default_config` declarations
- Lifecycle methods: `create`, `converge`, `setup`, `verify`, `destroy`
- Vagrantfile generation and templating
- SSH configuration parsing
- Version checking for Vagrant binary

**lib/kitchen/driver/vagrant_version.rb**
- Single constant `VAGRANT_VERSION = "0.15.0"`
- Used by gemspec for versioning

### Templates

**templates/Vagrantfile.erb**
- ERB template with access to driver's `config` hash
- Generates Vagrant 2.x compatible Vagrantfiles
- Supports multiple providers with conditional logic
- Handles customizations, networking, synced folders

### Build Files

**Rakefile**
- `rake cane` - Code quality metrics (complexity, style, documentation)
- `rake tailor` - Ruby style checking
- `rake stats` - Lines of code statistics
- `rake quality` - Runs all quality checks (default task)

**kitchen-vagrant.gemspec**
- Gem metadata and dependencies
- Runtime dependency: `test-kitchen ~> 1.0`
- Development dependencies: `cane`, `tailor`, `countloc`

## Configuration Options

### Required Configuration

**box** (String)
- Vagrant box name to use
- Default: `"opscode-#{platform_name}"`
- Example: `"opscode-ubuntu-14.04"`

### Optional Configuration

**box_url** (String)
- URL to download box if not locally installed
- Default: Computed from provider and platform
- Example: `"https://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_ubuntu-14.04_chef-provisionerless.box"`

**provider** (String)
- Vagrant provider: `virtualbox`, `vmware_fusion`, `vmware_workstation`, `rackspace`, `softlayer`
- Default: `ENV['VAGRANT_DEFAULT_PROVIDER']` or `"virtualbox"`

**customize** (Hash)
- Provider-specific VM customizations
- VirtualBox example: `{memory: 1024, cpuexecutioncap: 50}`
- VMware example: `{memory: 2048, numvcpus: 2}`

**network** (Array of Arrays)
- Network configurations
- Example: `[["forwarded_port", {guest: 80, host: 8080}], ["private_network", {ip: "192.168.33.33"}]]`

**synced_folders** (Array of Arrays)
- Synced folder configurations
- Format: `[[source, destination, options], ...]`
- Supports `%{instance_name}` variable substitution
- Example: `[["data/%{instance_name}", "/opt/instance_data"]]`

**pre_create_command** (String)
- Command to run before `vagrant up`
- Supports `{{vagrant_root}}` variable substitution
- Example: `"cp .vagrant_plugins.json {{vagrant_root}}/ && vagrant plugin bundle"`

**vagrantfile_erb** (String)
- Path to custom Vagrantfile ERB template
- Default: Uses built-in template
- Warning: Custom templates may reduce portability to other drivers

**vm_hostname** (String or false)
- Internal hostname for the VM
- Default: `"#{instance_name}.vagrantup.com"`
- Set to `false` to disable

**guest** (String)
- Guest OS type for Vagrant
- Example: `"ubuntu"`, `"windows"`

**username** (String)
- SSH username (overrides Vagrant default 'vagrant')

**ssh_key** (String)
- Path to private SSH key (overrides Vagrant's insecure default key)

**dry_run** (Boolean)
- If true, echoes Vagrant commands instead of executing
- Useful for debugging

## Development Commands

### Running Tests
```bash
# This gem doesn't have tests in the repository
# Tests would typically be in test/ or spec/ directory
```

### Code Quality Checks
```bash
rake cane      # Run quality metrics
rake tailor    # Check Ruby style
rake stats     # Display LOC statistics
rake quality   # Run all checks (default)
rake           # Same as 'rake quality'
```

### Building and Installing
```bash
gem build kitchen-vagrant.gemspec   # Build gem
gem install kitchen-vagrant-0.15.0.gem   # Install locally
```

### Publishing (Maintainers Only)
```bash
rake release   # Build, tag, and push to RubyGems (from bundler/gem_tasks)
```

## Related Repositories and Dependencies

### Direct Dependencies

**test-kitchen** (~> 1.0)
- Main framework this driver plugs into
- Provides base classes, SSH transport, and orchestration logic

### Test Kitchen Ecosystem

**Other Drivers:**
- `kitchen-ec2` - AWS EC2 instances
- `kitchen-docker` - Docker containers
- `kitchen-openstack` - OpenStack VMs
- `kitchen-digitalocean` - DigitalOcean droplets

**Provisioners:**
- `kitchen-chef` - Chef provisioner (built-in to Test Kitchen)
- `kitchen-puppet` - Puppet provisioner
- `kitchen-ansible` - Ansible provisioner
- `kitchen-salt` - Salt provisioner

**Verifiers:**
- `kitchen-inspec` - InSpec test framework
- `busser` - Plugin-based test runner (older approach)

### External Dependencies

**Vagrant** (>= 1.1.0)
- Must be installed as native package
- Download from: http://downloads.vagrantup.com/

**VirtualBox** (for default provider)
- Free, open-source virtualization
- Download from: https://www.virtualbox.org/wiki/Downloads

**VMware Fusion/Workstation** (optional)
- Commercial virtualization software
- Requires Vagrant VMware plugin (also commercial)

## Troubleshooting and Operations

### Common Issues

**"Vagrant 1.1.0 or higher is not installed"**
- Vagrant binary not in PATH
- Solution: Install Vagrant native package, uninstall old gem version if present

**"Detected an old version of Vagrant"**
- Vagrant version < 1.1.0
- Solution: Upgrade to Vagrant 1.1.0 or higher

**Box Download Failures**
- Default box URLs point to Opscode's S3 buckets
- May fail if URLs are outdated or boxes removed
- Solution: Specify custom `box_url` or use locally cached boxes

**Parallel Test Failures**
- Vagrant file locks conflict
- Driver prevents parallel create/destroy, but can still occur with manual operations
- Solution: Run create/destroy serially, other operations can be parallel

**VM Already Exists**
- Vagrant state file exists from previous failed run
- Solution: Manually run `vagrant destroy -f` in `.kitchen/kitchen-vagrant/<instance>/` or delete the directory

**SSH Connection Issues**
- `vagrant ssh-config` parsing failure
- Port forwarding conflicts
- Solution: Check VM is actually running, verify firewall rules, examine Vagrant logs

### Monitoring and Debugging

**Enable Debug Logging:**
```bash
kitchen create -l debug   # Verbose Test Kitchen output
```

**Debug Mode Features:**
- Outputs generated Vagrantfile contents
- Shows all Vagrant command output
- Displays SSH configuration parsing

**Check Vagrant State Directly:**
```bash
cd .kitchen/kitchen-vagrant/<instance-name>/
vagrant status           # Check VM state
vagrant ssh-config       # View SSH configuration
vagrant global-status    # See all Vagrant VMs
```

**Dry Run Mode:**
```yaml
driver:
  name: vagrant
  dry_run: true   # Echo commands without executing
```

### Manual Operations

**Access VM Shell:**
```bash
cd .kitchen/kitchen-vagrant/<instance-name>/
vagrant ssh
```

**Manually Destroy VM:**
```bash
cd .kitchen/kitchen-vagrant/<instance-name>/
vagrant destroy -f
```

**Inspect Generated Vagrantfile:**
```bash
cat .kitchen/kitchen-vagrant/<instance-name>/Vagrantfile
```

## Development Setup

### Prerequisites
- Ruby 1.9.3 or higher
- Bundler gem
- Vagrant >= 1.1.0
- VirtualBox (or other supported provider)

### Initial Setup
```bash
git clone https://github.com/test-kitchen/kitchen-vagrant.git
cd kitchen-vagrant
bundle install
```

### Making Changes

1. **Modify Ruby code** in `lib/kitchen/driver/`
2. **Run quality checks:** `rake quality`
3. **Test locally:** Install gem locally and test with a real `.kitchen.yml`
4. **Update version** in `lib/kitchen/driver/vagrant_version.rb`
5. **Update CHANGELOG.md** with changes

### Code Style Guidelines

- Follow existing Ruby style conventions
- Keep methods focused and small
- Add comments explaining WHY, not WHAT
- Use descriptive variable names
- Maintain backward compatibility when possible

## Usage Examples

### Basic .kitchen.yml
```yaml
---
driver:
  name: vagrant

provisioner:
  name: chef_solo

platforms:
  - name: ubuntu-14.04
  - name: centos-6.5

suites:
  - name: default
    run_list:
      - recipe[mycookbook::default]
```

### Advanced Configuration
```yaml
---
driver:
  name: vagrant
  provider: vmware_fusion
  network:
    - ["forwarded_port", {guest: 80, host: 8080}]
    - ["private_network", {ip: "192.168.33.33"}]
  synced_folders:
    - ["./data", "/opt/data"]
  customize:
    memory: 2048
    numvcpus: 2
  pre_create_command: cp .vagrant_plugins.json {{vagrant_root}}/

platforms:
  - name: ubuntu-14.04
    driver:
      box: custom-ubuntu-box
      box_url: https://example.com/boxes/ubuntu.box
      vm_hostname: testnode.example.com
```

### Common Test Kitchen Workflows

**Create VM and provision:**
```bash
kitchen create    # Boot VM
kitchen converge  # Run provisioner
```

**Run tests:**
```bash
kitchen verify    # Run test suite
```

**Full test cycle:**
```bash
kitchen test      # create + converge + verify + destroy
```

**Access VM:**
```bash
kitchen login     # SSH into VM
```

**Clean up:**
```bash
kitchen destroy   # Tear down VM
```

## History and Context

### Project Origin
- Created by Fletcher Nichol for Chef ecosystem testing
- Part of the Test Kitchen 1.0 rewrite (driver abstraction)
- Originally hosted under Opscode (now Chef Software)

### Evolution
- v0.11.x: Initial stable releases
- v0.12.0: Major refactor to ERB template approach
- v0.13.0: Support for Bento boxes (Opscode's minimal base boxes)
- v0.14.0: Added vm_hostname configuration
- v0.15.0: Added vagrant-softlayer plugin support

### Current Status
- Mature, stable driver
- Wide adoption in Chef community
- Maintained by Test Kitchen team
- Part of standard Test Kitchen workflow

## SendGrid/Twilio Context

This repository is a fork under the SendGrid organization, likely used for:
- Testing SendGrid infrastructure cookbooks
- CI/CD pipeline integration for infrastructure testing
- Development environment standardization

### SSC Team Usage

As part of the SSC (Secure Supply Chain) team's Buildkite infrastructure:
- May be used for testing Buildkite agent configurations
- Could be part of cookbook testing workflows
- Supports local development and testing of infrastructure changes

### Related SSC Infrastructure

Check these related repositories for context:
- Buildkite infrastructure repositories - CI/CD pipelines using Test Kitchen
- Jenkins configuration repositories (legacy) - Legacy CI systems
- Chef cookbook repositories that use Test Kitchen for testing

## Important Reminders

- **NEVER modify code logic** when adding comments - only add documentation
- All code in this repository is **production code** handling live traffic
- This is a **dependency** of other projects - breaking changes affect downstream users
- Test Kitchen is used in **CI/CD pipelines** - reliability is critical
- Vagrant version requirements are **strict** - don't relax them without testing
- The driver assumes Vagrant is **properly installed** - validation happens at runtime
- Parallel operations are **limited by design** due to Vagrant's locking behavior
- Each test instance gets an **isolated sandbox** - don't share state between instances

## Quick Reference

**Repository:** `sendgrid-ops/kitchen-vagrant`

**Main Entry Point:** `lib/kitchen/driver/vagrant.rb`

**Version File:** `lib/kitchen/driver/vagrant_version.rb`

**Template:** `templates/Vagrantfile.erb`

**Documentation:** `README.md` (user-facing), `CLAUDE.md` (this file, AI/developer)

**Changelog:** `CHANGELOG.md`

**Quality Commands:**
- `rake` - Run all quality checks
- `rake cane` - Quality metrics only
- `rake tailor` - Style checking only
- `rake stats` - LOC statistics

**Test Kitchen Commands:**
- `kitchen create` - Boot VM
- `kitchen converge` - Provision VM
- `kitchen verify` - Run tests
- `kitchen destroy` - Tear down VM
- `kitchen test` - Full cycle
- `kitchen login` - SSH access
