---
layout: post
title: "Splunk Server Setup Guide - Linux"
date: 2026-01-21
categories: notes
tags: [splunk, linux, security, siem]
---

# Splunk Server Setup Guide - Linux Environment

Complete guide for setting up Splunk Server with deployment server configuration.

## Configuration Variables

Update these at the top of the script:

DEPLOYMENT_SERVER_IP="167.172.73.162"
SPLUNK_ADMIN="admin"
SPLUNK_PASSWORD="admin123!"
TIMEZONE="Asia/Kuala_Lumpur"

## Installation Script

#!/bin/bash
set -e

# CONFIGURATION
DEPLOYMENT_SERVER_IP="167.172.73.162"
SPLUNK_ADMIN="admin"
SPLUNK_PASSWORD="admin123!"
TIMEZONE="Asia/Kuala_Lumpur"

SPLUNK_VERSION="10.0.2"
SPLUNK_BUILD="e2d18b4767e9"
SPLUNK_DEB="splunk-$SPLUNK_VERSION-$SPLUNK_BUILD-linux-amd64.deb"
SPLUNK_URL="https://download.splunk.com/products/splunk/releases/$SPLUNK_VERSION/linux/$SPLUNK_DEB"
SPLUNK_USER="splunk"
SPLUNK_HOME="/opt/splunk"

echo "=== Splunk Server Installation ==="

echo "[1/10] Setting timezone..."
sudo timedatectl set-timezone $TIMEZONE

echo "[2/10] Updating system..."
sudo apt update && sudo apt upgrade -y

echo "[3/10] Installing tools..."
sudo apt install -y wget net-tools ufw vim

echo "[4/10] Downloading Splunk..."
cd /tmp
wget -O $SPLUNK_DEB "$SPLUNK_URL"

echo "[5/10] Installing Splunk..."
sudo dpkg -i $SPLUNK_DEB
rm $SPLUNK_DEB

echo "[6/10] Starting Splunk..."
sudo $SPLUNK_HOME/bin/splunk start --accept-license --answer-yes

echo "[7/10] Configuring firewall..."
sudo ufw allow 8000/tcp
sudo ufw allow 9997/tcp
sudo ufw allow 8089/tcp
sudo ufw reload

echo "[8/10] Creating splunk user..."
sudo useradd -m -s /bin/bash $SPLUNK_USER || true

echo "[9/10] Setting ownership..."
sudo $SPLUNK_HOME/bin/splunk stop
sudo chown -R $SPLUNK_USER:$SPLUNK_USER $SPLUNK_HOME
sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk enable boot-start -user $SPLUNK_USER

echo "[10/10] Enabling deployment server..."
sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk enable deploy-server -auth $SPLUNK_ADMIN:$SPLUNK_PASSWORD
sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk set deploy-poll $DEPLOYMENT_SERVER_IP:8089 -auth $SPLUNK_ADMIN:$SPLUNK_PASSWORD

sudo -u $SPLUNK_USER $SPLUNK_HOME/bin/splunk restart

echo "Installation Complete!"
echo "Web: http://YOUR_IP:8000"
echo "Username: $SPLUNK_ADMIN"
