# Mobile App

K8s Disrupter is a simple react native application built using [expo.io](https://expo.io/)

> [Original repository](https://github.com/danacr/k8s-disrupter), I also added it as a [submodule](../k8s-disrupter/README.md), to this repository.

It uses 2 methods to trigger HTTP post requests to the [disrupter-server](../k8s-disrupter)

Shaking:

```js
name = Constants.deviceName;
id = Constants.installationId;
async componentWillMount() {
ShakeEventExpo.addListener(() => {
    fetch("https://disrupt.mad.md/", {
    method: "POST",
    body: JSON.stringify({ Name: this.name, ID: this.id })
    }).catch();
    this.runAnimation();
    Vibration.vibrate(500);
});
}
```

Clicking an awesome button:

```js
<AwesomeButton
  onPress={() => {
    fetch("https://disrupt.mad.md/", {
      method: "POST",
      body: JSON.stringify({ Name: this.name, ID: this.id })
    }).catch();
    this.runAnimation();
    Vibration.vibrate(500);
  }}
>
  <Text
    style={{
      color: "#fff",
      fontSize: 20,
      padding: 10
    }}
  >
    Trigger Disruption
  </Text>
</AwesomeButton>
```

## That's it, it's all it does :)

Next: [Disrupeter Server](02-disrupter-server.md)
