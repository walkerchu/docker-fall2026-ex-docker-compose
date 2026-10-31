
# Exercise 2.11: Containers in Development

## Project Summary

This project is a multi-container setup built for a development and testing environment. It separates the web application logic from the quality assurance (QA) automation tools.

The primary goals and details of this repository are:

- **Exercise Submission**: This project is an answer submission for Exercise 2.11 in Chapter 3 of the DevOps with Docker (Fall 2026) course by the University of Helsinki.

- **Remote Repository**: The full codebase and configuration history are stored in the public remote repository:
    > https://github.com/walkerchu/multi-containers

- **What We Did**: We built a fully containerized development environment that links multiple separate containers together. This allows a developer to write code in the web service while a QA tester runs isolated browser validation suites in another container at the same time.

## Background and Architecture

As part of the submission of Exercise 2.11, we have created a fully blown containerized development environment. Instead of forcing all software into one heavy container or installing tools on the host system, we separated the components using Docker Compose:

1. Multi-Container Connection: The setup follows the official Microsoft VS Code guide for advanced containers: https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers. This architecture allows VS Code windows to connect to different individual containers inside the same network.
2. Environment Separation:
    - Node.js Environment: The web application runs natively on Node.js inside its own container service.
    - Isolated QA Environment: Playwright is installed only inside the specialized QA container. _The local Windows Subsystem for Linux (WSL) host environment does not have Playwright installed._ This keeps the host system clean and guarantees the tests run in an identical environment every time.
3. Context Preservation: The configuration mounts the root directory so that the containers can naturally see the parent `.git` directory. This preserves Git context and allows SSH credential forwarding to work smoothly.

## Folder Structure

Below is the project directory tree. It shows the layout of our multi-container system and points out the most important configuration files:

```text
.
├── .devcontainer/
│   └── devcontainer.json       <-- Configures the VS Code QA development box environment
├── web/
│   ├── docker-compose.yaml     <-- Defines the Express.js backend service
│   ├── package.json            <-- Web application dependencies (Express)
│   └── server.js               <-- Backend entrypoint with health diagnostics
├── qa/
│   ├── health.spec.js          <-- Playwright automated test scripts
│   ├── package.json            <-- QA framework dependencies (@playwright/test)
│   └── playwright.config.js    <-- Playwright configuration file
├── .gitignore
└── docker-compose.yaml         <-- Root orchestration file that includes sub-compose configurations
```

## How the Environments Interact

This project connects three different layers together: local Windows Subsystem for Linux (WSL) host, the Web application container, and the QA testing container. 

They interact with each other through the following workflow:

```
┌────────────────────────────────────────────────────────┐
│                      WSL HOST                          │
│  - Holds the project files (codebase and .git)         │
│  - Runs the main 'docker compose' commands             │
└──────────────────────────┬─────────────────────────────┘
│
│ Mounts code & shares network
▼
┌────────────────────────────────────────────────────────┐
│                 SHARED DOCKER NETWORK                  │
│                                                        │
│   ┌────────────────────┐      Sends HTTP requests      │
│   │    QA CONTAINER    ├──────────────────────────┐    │
│   │                    │                          │    │
│   │ - Runs Playwright  │                          ▼    │
│   │   test scripts     │                ┌───────────────────┐
│   └────────────────────┘                │   WEB CONTAINER   │
│                                         │                   │
│                                         │ - Runs the live   │
│                                         │   Express app     │
│                                         └───────────────────┘
└────────────────────────────────────────────────────────┘
```

### 1. The WSL Host (The Controller)
* The codebase, `.git` tracking history, and configuration files are stored entirely on the local WSL environment.
* No local installation of Node.js or Playwright is required on the host system. The WSL host is utilized solely to execute commands such as `docker compose up` and `docker compose exec` to manage the environment.

### 2. The Shared Docker Network (The Bridge)
* A private virtual network is automatically created by Docker Compose when the services are started.
* This network serves as a secure bridge, allowing containers to communicate directly using service names (such as `http://web:8000`) instead of public IP addresses.

### 3. The Web Container (The Application)
* The live Express.js web server is hosted inside this dedicated application container. 
* The container remains active to listen for incoming traffic, processing requests and returning health status diagnostics without any direct dependency on the QA tools.

### 4. The QA Container (The Tester)
* The testing architecture, including the Playwright framework and required browser engines, is kept isolated inside the QA container.
* When a test is triggered from the terminal, automated HTTP requests are sent from the QA container across the shared network to the Web container, responses are validated, and the final test results are displayed back on the terminal.


## How to Start the Playwright Test

Follow these steps to bring up the environment and run the automated health tests:

**Step 1: Start the Containers**

Open your terminal in the root folder (where the main docker-compose.yaml is located) and start all container services in detached mode:
```bash
docker compose up -d
```

**Step 2: Run the Playwright Test Suite**

Execute the automated test script inside the running QA container using the exec command:
```bash
docker compose exec playwright-runner npx playwright test --reporter=list
```

**Step 3: Stop the Environment**

When you are done testing and developing, you can stop the containers safely:
```bash
docker compose stop
```

Last Modified: October 31, 2026

