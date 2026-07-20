# AWS WireGuard VPN Setup Guide

This document explains the complete deployment process for the self-hosted WireGuard VPN on AWS.

## Deployment Overview

The infrastructure consists of:

- Custom Amazon VPC
- Public subnet
- Internet Gateway
- Custom route table
- Security Group
- Ubuntu EC2 instance
- WireGuard VPN server
- Windows WireGuard client
