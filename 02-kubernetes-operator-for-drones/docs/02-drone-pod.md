# Drone Pod

## Controlling the drone using [Gobot](https://github.com/hybridgroup/gobot)

> [Original repository](https://github.com/danacr/drone-pod), I also added it as a submodule, to this repository.

We need to create a small script where we detect on which drone the node is attached to and then take off.

```go
func main() {

	droneip := getDroneIP()
	drone := tello.NewDriver(droneip, "8888")
	work := func() {

		drone.On(tello.ConnectedEvent, func(data interface{}) {
			fmt.Println("Connected")
		})
		drone.TakeOff()
	}
	robot := gobot.NewRobot("tello",
		[]gobot.Connection{},
		[]gobot.Device{drone},
		work,
	)

	robot.Start()
}
```

> Note: you will see that there is a `replace` in the `go.mod` file, but it is [no longer needed](https://github.com/hybridgroup/gobot/pull/718).

```go
replace gobot.io/x/gobot => github.com/danacr/gobot v1.14.1-0.20191123165846-128782c85ca5
```

In order to detect which drone we need to start, we'll use a simple environment variable, since the Drone IPs are static:

```go
func getDroneIP() string {
	var droneip string
	switch node := os.Getenv("NODE"); node {
	case "pine00":
		droneip = "192.168.86.250"
	case "pine01":
		droneip = "192.168.86.251"
	case "pine02":
		droneip = "192.168.86.252"
	}
	return droneip
}
```

Last but not least, I've noticed that the drones fail to land when the pod is terminated, so we catch the `SIGTERM` and land before exiting:

```go
	// capture sigterm and land drone
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGTERM)
	go func() {
		for sig := range c {
			fmt.Println("Need to die, but I must land the drone first", sig)
			drone.Land()
			os.Exit(0)
		}
	}()
```

Next: [Drone Controller](03-drone-controller.md)
