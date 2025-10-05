# HAComfoAirMQTT Installation Guide for Pi OS

This guide provides installation instructions for hacomfoairmqtt on modern Raspberry Pi OS (Bullseye/Bookworm).

## Prerequisites

Install required system packages:

```bash
sudo apt update
sudo apt install -y python3-full python3-pip python3-venv git
```

## Installation Steps

### 1. Create Project Directory

Create a project directory in your home folder:

```bash
cd ~
mkdir -p hacomfoair
cd hacomfoair
```

### 2. Clone Repository

```bash
git clone https://github.com/heikil/hacomfoairmqtt.git .
```

### 3. Create Virtual Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 4. Install Python Dependencies

```bash
pip install python-etcd paho-mqtt pyserial
```

### 5. Configure the Service

First, look up the SerialPort device ID for integration in config.ini:

```bash
ls /dev/serial/by-id
```

Copy the configuration template and edit it:

```bash
cp src/config.ini.dist src/config.ini
nano src/config.ini
```

In the nano editor, change the following settings:
- **SerialPort**: Use the device ID from the previous step (e.g., `/dev/serial/by-id/usb-XXXXXX`)
- **MQTT-Server**: Your MQTT broker address
- **User**: MQTT broker username
- **Password**: MQTT broker password

Save and exit: `Ctrl + O`, then `Ctrl + X`

### 6. Create Systemd Service

Create the service file:

```bash
sudo nano /etc/systemd/system/ca350.service
```

Add the following content (adjust username if not 'h'):

```ini
[Unit]
Description=ComfoAir CA350 MQTT Service
After=network.target

[Service]
Type=simple
User=h
WorkingDirectory=/home/h/hacomfoair
ExecStart=/home/h/hacomfoair/.venv/bin/python /home/h/hacomfoair/src/ca350.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Save and exit: `Ctrl + O`, then `Ctrl + X`

### 7. Enable and Start Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable ca350.service
sudo systemctl start ca350.service
```

### 8. Check Service Status

```bash
sudo systemctl status ca350.service
```

## Troubleshooting

View service logs:

```bash
sudo journalctl -u ca350.service -f
```

Restart the service:

```bash
sudo systemctl restart ca350.service
```

## Important Notes

- Always activate the virtual environment before installing packages: `source .venv/bin/activate`
- Never use `sudo` with `pip` when working inside a virtual environment
- All paths in the systemd service must point to your home directory and virtual environment