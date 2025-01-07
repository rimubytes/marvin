---
layout: post
title: "Container Insights Part 2: Understanding netns for Network Isolation"
description: "Explore Network Namespaces (netns) – a key component of container networking. Learn how to create isolated network environments for your containers using Go code examples."
comments: true
tags: networking nets containerization
excerpt_separator: <!--more-->
---

In the dynamic realm of containerized applications, robust network isolation is paramount. Network Namespaces (netns), a cornerstone of container networking, provide this crucial layer of separation. By encapsulating a complete network stack – interfaces, IP addresses, routing tables – within a distinct namespace, netns empowers containers to operate with their own independent network environments.
<!--more-->

In this installment, we delve into network namespaces (netns). If you’ve ever wondered how containers achieve network isolation or how processes are assigned unique networking environments, this post is for you. By exploring the concepts and walking through practical examples, we’ll uncover how `netns` plays a critical role in containerized systems.

## What is a Network Namespace `(netns)`?

In Linux, a network namespace (netns) provides an isolated network stack for a group of processes. This includes network interfaces, IP addresses, routing tables, and more. Each namespace operates independently, creating the foundation for the network isolation containers require.

For example:

- Containers in Docker or Kubernetes are assigned their own netns.
- Processes in separate netns can have the same IP addresses without conflicts.

By default, all processes share the default network namespace of the host, but tools like Docker, Podman, and Kubernetes manage these namespaces dynamically to provide isolation.

### Why Use `netns`?

1. **Network Isolation**: Separate containers or processes from the host’s networking to prevent interference.
2. **Custom Networking**: Create private or shared networks for container communication.
3. **Enhanced Security**: Limit a container’s access to the host’s networking capabilities.

### Hands-On: Managing `netns` with Go

Let’s explore how to create, join, and manage network namespaces programmatically in Go.

1. **Setting Up a New Network Namespace**

The setupNewNetworkNamespace function creates a new netns and saves it as a file in /tmp/net-ns. This is achieved through:

Unsharing the current network namespace using syscall.Unshare().
Bind-mounting the new namespace to a file for persistence.
Restoring the process to the original namespace.

Key snippet:

```go
func setupNewNetworkNamespace(processID int) {
    nsMount := fmt.Sprintf("%s/%d", getNetNsPath(), processID)

    // Create a new namespace
    if err := syscall.Unshare(syscall.CLONE_NEWNET); err != nil {
        log.Fatalf("Unshare system call failed: %v\n", err)
    }

    // Bind mount the namespace to a file
    if err := syscall.Mount("/proc/self/ns/net", nsMount, "bind", syscall.MS_BIND, ""); err != nil {
        log.Fatalf("Mount system call failed: %v\n", err)
    }

    log.Printf("New network namespace created for process: %d\n", processID)
}
```

2. **Joining a Network Namespace**

The joinContainerNetworkNamespace function allows a process to switch to an existing namespace. This is particularly useful when multiple containers or processes need to share the same network stack.

Key snippet:

```go
func joinContainerNetworkNamespace(processID int) error {
    nsMount := fmt.Sprintf("%s/%d", getNetNsPath(), processID)
    fd, err := unix.Open(nsMount, unix.O_RDONLY, 0)
    if err != nil {
        log.Printf("Unable to open network namespace file: %v\n", err)
        return err
    }
    defer unix.Close(fd)

    // Switch to the specified namespace
    if err := unix.Setns(fd, unix.CLONE_NEWNET); err != nil {
        log.Printf("Setns system call failed: %v\n", err)
        return err
    }
    return nil
}
```

