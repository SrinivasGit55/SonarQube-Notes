# SonarQube Notes

A structured collection of my SonarQube learning notes, DevSecOps concepts, practical installation steps, CI/CD integration examples and troubleshooting points.

## Learning Progress

### Introduction, Architecture and DevSecOps Concepts

[Read SonarQube Theory Notes](https://github.com/SrinivasGit55/SonarQube-Notes/blob/main/SonarQube-Notes.md)

**Covered topics:**

- Introduction to SonarQube and its role in DevSecOps
- Shift Left Security and secure software development
- SAST, DAST, SCA and SBOM concepts
- SonarQube architecture: Server, Scanner, Database and Elasticsearch
- Bugs, Vulnerabilities, Security Hotspots and Code Smells
- Quality Profiles and Quality Gates
- New Code versus Overall Code analysis
- Code Coverage and JaCoCo integration
- Maven-based code analysis
- Jenkins integration and Quality Gate checks
- SonarQube in a CI/CD workflow
- Practical scenarios and troubleshooting

### Practical Installation and Integration

[Read SonarQube Installation and Integration Notes](https://github.com/SrinivasGit55/SonarQube-Notes/blob/main/SonarQube-Installation-and-Integration.md)

**Covered topics:**

- SonarQube Server installation on Ubuntu 24.04
- SonarQube Server installation on AWS Amazon Linux 2023
- Java 21 installation and configuration
- PostgreSQL installation and database configuration
- Linux kernel prerequisites for Elasticsearch
- SonarQube user, directory and permission configuration
- SonarQube service configuration using systemd
- Accessing SonarQube through port 9000
- Maven project analysis using SonarScanner for Maven
- JaCoCo code coverage report generation
- Jenkins and SonarQube integration
- Jenkins credentials and SonarQube authentication
- Quality Gate integration using `waitForQualityGate`
- Webhook configuration
- Installation, connectivity and integration troubleshooting

## Learning Approach

These notes are organized into theory and practical implementation sections. The focus is on understanding SonarQube architecture, code quality analysis, security concepts and how automated code analysis integrates with Jenkins and Maven in a CI/CD pipeline.

The practical examples cover Linux-based installation, database configuration, project analysis, Quality Gate enforcement and troubleshooting to build an understanding of how SonarQube supports code quality and security in real-world DevOps workflows.
