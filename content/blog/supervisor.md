---
title: 'Process management on Amazon linux Al2023 with Supervisor'
date: 2023-01-25T10:10:38+02:00
draft: false
---

#### Introduction

Have you ever found yourself wrestling with managing processes on your server efficiently? Enter Supervisor – a robust and adaptable process control system designed for Unix-like systems. In this tutorial, we'll walk you through the process of installing and setting up Supervisor on Amazon Linux 2023. Whether you're a developer, sysadmin, or simply someone eager to enhance your server management skills, this guide is tailored for you.

#### What is Supervisor?

Supervisor is a process control system that simplifies the management of processes, applications, and task scheduling on Unix-like operating systems. Its user-friendly interface and powerful features make it an ideal choice for those seeking to streamline and enhance their server management capabilities.

#### Prerequisites

Before diving into the installation process, make sure you have:

- An instance of Amazon Linux 2023
- Access to the server with administrative privileges
- This tutorial assumes we are using the "deployer" user for deploying Supervisor
- This tutorial showcases installing pip packages on a custom path

#### Step1: Python3 and Pip

Make sure that Python3 and pip is installed on the machine. You can install pip as follows

```sh
dnf install pip -y
```

#### Step2: Install supervisor

Now, let's install Supervisor using a handy script. Save the following script as install.sh, change its owner to "deployer," and execute it.

```sh
sudo chown -R deployer:deployer install.sh
```

The install script:

```sh
#!/bin/bash
set -e

name="supervisor"
supervisor_version="4.2.5"
install_path_prefix="/opt"
package_name="${name}-${supervisor_version}"
script_dir="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"; # The directory containing this script

# Verify Python and pip
su deployer -c "python -V 2>&1 | grep -q -e "Python 3\."' 2>/dev/null"
exitValue=$?
if [ $exitValue -eq 0 ]; then
    echo "Python3 has not been installed"
    exit 1
else
    echo "Found python 3"
    which python
    python -V
fi

su deployer -c "pip --version 2>&1 | grep -q -e "pip 2"' 2>/dev/null"
exitValue=$?
if [ $exitValue -eq 0 ]; then
    echo "pip has not been installed"
    exit 1
else
    echo "Found pip 3"
    which pip
    pip --version
fi

# Create directories needed for the installation
mkdir -p ${install_path_prefix}
mkdir -p ${install_path_prefix}/${package_name}
mkdir -p ${install_path_prefix}/${package_name}/etc
mkdir -p /var${install_path_prefix}/${package_name}

# Install supervisor4
echo "Installing supervisor ${supervisor_version}"
export PYTHONUSERBASE=${install_path_prefix}/${name}
pip install supervisor==${supervisor_version}

# Put the supervisord.conf file in place
su deployer -c "cp ${script_dir}/supervisord.conf ${install_path_prefix}/${package_name}/etc"

# Substitute PORT in supervisord.conf
if [ "${install_path_prefix}" = "/opt" ]; then
    PORT=9001
else
    PORT=9003
fi
su deployer -c "sed -i \'s|PORT|${PORT}|\' ${install_path_prefix}/${package_name}/etc/supervisord.conf "

# Create the symlink
su deployer -c "ln -sfn ${install_path_prefix}/${package_name} ${install_path_prefix}/${name}"
```

supervisor config file is as follows, Since we have changed the default pip installation path by setting PYTHONUSERBASE environmental variable, same variable need to be set on the config file.
If you wish use the default configuration, do like that.

Instead of

```sh
su deployer -c "cp ${script_dir}/supervisord.conf ${install_path_prefix}/${package_name}/etc"
```

run this

```sh
echo_supervisord_conf > ${install_path_prefix}/${package_name}/etc/supervisord.conf
```

Here you can find an example on a custom config file

```
; Sample supervisor config file.
;
; For more information on the config file, please see:
; http://supervisord.org/configuration.html
;
; Notes:
;  - Shell expansion ("~" or "$HOME") is not supported.  Environment
;    variables can be expanded using this syntax: "%(ENV_HOME)s".
;  - Comments must have a leading space: "a=b ;comment" not "a=b;comment".

[unix_http_server]
file=/varPREFIX/supervisor/supervisor.sock   ; (the path to the socket file)

[inet_http_server]         ; inet (TCP) server disabled by default
port=0.0.0.0:PORT                ; (ip_address:port specifier, *:port for all iface)

[supervisord]
logfile=/varPREFIX/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log)
logfile_maxbytes=50MB        ; (max main logfile bytes b4 rotation;default 50MB)
logfile_backups=10           ; (num of main logfile rotation backups;default 10)
loglevel=info                ; (log level;default info; others: debug,warn,trace)
pidfile=/tmp/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
nodaemon=false               ; (start in foreground if true;default false)
minfds=1024                  ; (min. avail startup file descriptors;default 1024)
minprocs=200                 ; (min. avail process descriptors;default 200)
user=deployer                 ; (default is current user, required if root)
childlogdir=/varPREFIX/supervisor/            ; ('AUTO' child log dir, default $TEMP)
environment=PYTHONUSERBASE="/opt/supervisor"
strip_ansi=true              ; (strip ansi escape codes in logs; def. false)

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=http://127.0.0.1:PORT
environment=PYTHONUSERBASE="/opt/supervisor"

[include]
files = supervisor.d/*.ini
```

This script installs Supervisor version 4.2.5 and sets up the necessary directories and configurations. Customize it according to your preferences.

#### Starting supervisor on each reboot

Once Supervisor is installed for the "deployer" user under /opt, ensure it runs on every start and reboot by adding Supervisor as a systemd unit.

```sh
echo "Starting supervisord..."
systemd_path="/etc/systemd/system"
sudo cp ${script_dir}/supervisord.service ${sysemd_path}/supervisord.service
sudo chown deployer:deployer ${sysemd_path}/supervisord.service
sudo chmod 755 ${sysemd_path}/supervisord.service
sudo systemctl daemon-reload
sudo systemctl enable supervisord
sudo systemctl start supervisord
echo "Done starting supervisord"
```

You also need run the supervisor systemd service as "deployer" user. It can be done in system unit file (supervisord.service) as follows

```
[Unit]
Description=A service that starts supervisord.
After=network.target

[Service]
Type=forking
ExecStart=/bin/bash --login -c '/opt/supervisor/bin/supervisord -c /opt/supervisor/etc/supervisord.conf'
User=deployer
Group=deployer
Restart=always

[Install]
WantedBy=multi-user.target

```

This script creates a systemd service for Supervisor, ensuring it starts on boot and is managed by the "deployer" user.

Now, you've successfully installed Supervisor on Amazon Linux Al2023 and configured it to run seamlessly, supercharging your process management capabilities. Feel free to explore Supervisor's extensive documentation for advanced features and customization options. Happy supervising! 🚀
