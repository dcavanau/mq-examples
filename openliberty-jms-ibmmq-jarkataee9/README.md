# Open Liberty - JMS & IBM MQ With SSL

A sample application that uses Open Liberty to connect to IBM MQ to enqueue & dequeue messages.

## Setup

For this project you'll need to install the following requirements:

- JDK 17 (or later) - E.g. [Temurim](https://adoptium.net/installation/linux)
- Latest Maven, it must be Java 17 compatible - <https://maven.apache.org/install.html>
- Podman

## Running it

Pull and start the IBM MQ docker container:

```sh
podman pull icr.io/ibm-messaging/mq:latest

podman run --env LICENSE=accept \
  --env MQ_QMGR_NAME=QM1 \
  --publish 1414:1414 \
  --detach \
  --env MQ_APP_PASSWORD=passw0rd \
  --name QM1 icr.io/ibm-messaging/mq:latest
```

> _Since 9.2.5.0 new MQ images will be published to IBM's registry._

Install the dependencies and start OpenLiberty:

```sh
mvn install
mvn liberty:dev
```

The liberty server will start and you will see something similar to the following:

```sh
[INFO] CWWKM2015I: Match number: 1 is [11/26/24, 10:54:25:973 EST] 00000030 com.ibm.ws.kernel.feature.internal.FeatureManager            A CWWKF0011I: The defaultServer server is ready to run a smarter planet. The defaultServer server started in 7.084 seconds..
[INFO] ************************************************************************
[INFO] *    Liberty is running in dev mode.
[INFO] *        Automatic generation of features: [ Off ]
[INFO] *        h - see the help menu for available actions, type 'h' and press Enter.
[INFO] *        q - stop the server and quit dev mode, press Ctrl-C or type 'q' and press Enter.
[INFO] *
[INFO] *    Liberty server port information:
[INFO] *        Liberty server HTTP port: [ 9080 ]
[INFO] *        Liberty server HTTPS port: [ 9443 ]
[INFO] *        Liberty debug port: [ 7777 ]
[INFO] ************************************************************************
[INFO] Source compilation was successful.
[INFO] Tests compilation was successful.
[INFO] [AUDIT   ] CWWKT0017I: Web application removed (default_host): http://10.0.0.26:9080/libertymq/
[INFO] [AUDIT   ] CWWKZ0009I: The application libertymq has stopped successfully.
[INFO] [AUDIT   ] CWWKT0016I: Web application available (default_host): http://10.0.0.26:9080/libertymq/
[INFO] [AUDIT   ] CWWKZ0003I: The application libertymq updated in 0.363 seconds.
```

Once the server is running, send a message to the queue:

```sh
curl -X POST -d 'msg=test message' http://localhost:9080/libertymq/api/enqueue
```

In in Liberty server window, you should see a message similar to this:

```sh
[INFO] Message enqueued.
[INFO] MDB received: test message
```

## Versions

Implementation / tests performed with the latest/LTS versions of all components:

- openjdk 17.0.3 2022-04-19
- Jakarta EE 9.1
- Apache Maven 3.8.6
- OpenLibery 22.0.0.7
- IBM MQ & Resource Adapter 9.3.0.0

ℹ️ For convenience I'm versioning the adapter here. But you should get your specific version from [Fix Central](https://www.ibm.com/support/fixcentral/), following guidelines such as in [this procedure](https://www.ibm.com/docs/en/ibm-mq/9.3?topic=adapter-installing-resource-in-liberty) for your particular version.

## Sources

```text
https://developer.ibm.com/tutorials/mq-connect-app-queue-manager-containers/
https://www.ibm.com/docs/en/was-liberty/nd?topic=dmal-deploying-jms-applications-liberty-use-mq-messaging-provider
https://developer.ibm.com/tutorials/mq-develop-mq-jms/
https://aguibert.github.io/openliberty-cheat-sheet/
```
