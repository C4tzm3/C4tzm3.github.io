---
layout: post
title: "Splunk Universal Forwarder Setup - Linux"
date: 2026-01-21
categories: notes
tags: [splunk, linux, forwarder]
---

# Splunk Universal Forwarder Setup - Linux

## Configuration

DEPLOYMENT_SERVER="167.172.73.162:8089"
SPLUNK_RECEIVER="167.172.73.162:9997"
TIMEZONE="Asia/Kuala_Lumpur"

## Installation Script

#!/bin/bash
set -e

DEPLOYMENT_SERVER="167.172.73.162:8089"
SPLUNK_RECEIVER="167.172.73.162:9997"
TIMEZONE="Asia/Kuala_Lumpur"

SPLUNK_VERSION="9.3.2"
SPLUNK_BUILD="d8bb32809498"
UF_DEB="splunkforwarder-$SPLUNK_VERSION-$SPLUNK_BUILD-linux-2.6-amd64.deb"
UF_URL="https://download.splunk.com/products/universalforwarder/releases/$SPLUNK_VERSION/linux/$UF_DEB"
SPLUNK_HOME="/opt/splunkforwarder"

echo "=== Universal Forwarder Installation ==="

sudo timedatectl set-timezone $TIMEZONE
sudo apt update && sudo apt upgrade -y

cd /tmp
wget -O $UF_DEB "$UF_URL"
sudo dpkg -i $UF_DEB
rm $UF_DEB

sudo $SPLUNK_HOME/bin/splunk start --accept-license --answer-yes
sudo $SPLUNK_HOME/bin/splunk set deploy-poll $DEPLOYMENT_SERVER -auth admin:changeme
sudo $SPLUNK_HOME/bin/splunk add forward-server $SPLUNK_RECEIVER -auth admin:changeme
sudo $SPLUNK_HOME/bin/splunk enable boot-start
sudo $SPLUNK_HOME/bin/splunk restart

echo "Installation Complete!"
