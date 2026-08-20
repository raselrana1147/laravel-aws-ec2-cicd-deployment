# Laravel Project Deployment Guide (EC2 + GitHub Actions CI/CD)

Complete, repeatable deployment guide for hosting a Laravel application on an AWS EC2 instance with Nginx, MySQL, and automated deployment via GitHub Actions.

## Table of Contents
* [Architecture Overview](#architecture-overview)
* [Prerequisites](#prerequisites)
* [Part 1 — EC2 Server Setup](#part-1--ec2-server-setup)
* [Part 2 — Install Required Packages](#part-2--install-required-packages)
* [Part 3 — Database Setup](#part-3--database-setup)
* [Part 4 — Clone and Configure the Laravel Project](#part-4--clone-and-configure-the-laravel-project)
* [Part 5 — Configure Nginx](#part-5--configure-nginx)
* [Part 6 — Set Up GitHub Actions CI/CD](#part-6--set-up-github-actions-cicd)
* [Part 7 — SSL Setup (Optional)](#part-7--ssl-setup-optional)
* [Security Checklist](#security-checklist)
* [Troubleshooting](#troubleshooting)
* [Useful Commands Reference](#useful-commands-reference)

---

## Architecture Overview

```text
Developer pushes to GitHub (main branch)
              ↓
GitHub Actions triggers workflow
              ↓
SSH connects to EC2 → pulls code / syncs files
              ↓
Runs: composer install → migrate → cache
              ↓
Nginx + PHP-FPM serve the app
              ↓
App connects to MySQL (RDS or local on EC2)
