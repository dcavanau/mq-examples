# Open Liberty - JMS & IBM MQ With SSL

A sample application that uses Open Liberty to connect via SSL to IBM MQ to enqueue & dequeue messages. This example uses *1-way authentication*.

The default behavior for the MQ Client application is to ask for the authentication of the
MQ queue manager, this is called *1-way authentication*. This is accomplished by specifying **SSLCAUTH(OPTIONAL)** in the definition of the channel.

The method of *2-way authentication* is when additionally, the MQ queue manager asks for the authentication of the MQ Client application. This is accomplished by specifying **SSLCAUTH(REQUIRED)** in the definition of the channel.

Self-signed certificates are used in this example. Self-signed TLS certificates are suitable for personal use or for applications that are used internally within an organization. Self-signed certificates are **NOT** suitable for software used by external users or high-risk or Production environments because the certificates cannot be verified with a Certification
Authority (CA).

## Setup

For this project you'll need to install the following requirements:

- JDK 17 (or later) - E.g. [Temurim](https://adoptium.net/installation/linux)
- Latest Maven, it must be Java 17 compatible - <https://maven.apache.org/install.html>
- Podman

## Set Up MQ Server keystore and certificate

1. Pull and start the IBM MQ docker container:

   ```sh
   podman volume create qmdata

   podman pull icr.io/ibm-messaging/mq:latest

   podman run -ti \
     --entrypoint=/bin/bash \
     --volume qmdata:/mnt/mqm \
     icr.io/ibm-messaging/mq:latest
   ```

2. Change to the directory that is mounted on the volume to create digital certificates for the server

   ```sh
   cd /mnt/mqm
   mkdir -p MQServer/certs
   cd MQServer/certs
   ```

3. Create a key database (also called the keyStore or certificate store), then add and stash the password for it

   ```sh
   runmqakm -keydb -create -db key.p12 -pw k3ypassw0rd -type pkcs12 -expire 1000 -stash
   ```

4. Check what has been created so far

   ```sh
   ls
    > key.p12  key.sth
   ```

5. Check the contents of the keyStore

   ```sh
   runmqakm -cert -list all -db key.p12 -stashed
   > No certificates were found.
   ```

6. Create a self signed certificate

   ```sh
   runmqakm -cert -create -db key.p12 -label ibmwebspheremqqm1 -dn "cn=qm,o=ibm,c=us" -size 2048 -default_cert yes -stashed
   ```

7. Check the contents of the keyStore

   ```sh
   runmqakm -cert -list all -db key.p12 -stashed
   > Certificates found
     * default, - personal, ! trusted, # secret key
     - ibmwebspheremqqm1
   ```

8. Extract public-key for client to communicate with the Queue Manager

   ```sh
   runmqakm -cert -extract -db key.p12 -stashed -label ibmwebspheremqqm1 -target QM1.cert
   ```

9. Check what has been created so far

   ```sh
   ls
    > QM1.cert  key.p12  key.sth
   ```

10. Exit the container. The certificate data will be safe in the volume

    ```sh
    exit
    podman container ls --all
    > CONTAINER ID  IMAGE                           COMMAND     CREATED         STATUS                     PORTS                                   NAMES
      1d57f8d01b9b  icr.io/ibm-messaging/mq:latest              24 minutes ago  Exited (0) 27 seconds ago  1414/tcp, 9157/tcp, 9415/tcp, 9443/tcp  great_einstein
    podman container rm 1d57f8d01b9b
    > 1d57f8d01b9b
    ```

## MQ Queue Manager configuration in container

1. Start the container

   ```sh
   podman run \
     --env LICENSE=accept \
     --env MQ_QMGR_NAME=QM1 \
     --volume qmdata:/mnt/mqm \
     --publish 1414:1414 \
     --publish 1419:1419 \
     --publish 8443:9443 \
     --detach \
     --env MQ_APP_PASSWORD=passw0rd \
     --env MQ_TLS_KEYSTORE=/mnt/mqm/MQServer/certs/key.p12 \
     --env MQ_TLS_PASSPHRASE=k3ypassw0rd \
     --name QM1 \re
     ibmcom/mq:latest
   ```

2. Enter the container

   ```sh
   podman exec -ti QM1 /bin/bash
   ```

3. Start the MQSC interface for the queue manager

   ```sh
   runmqsc QM1
   > 5724-H72 (C) Copyright IBM Corp. 1994, 2021.
     Starting MQSC for queue manager QM1.
   ```

4. Issue the commands to define the SSL channel

   ```sh
   DEFINE CHANNEL('DEV.SSL.SVRCONN') CHLTYPE(SVRCONN) TRPTYPE(TCP) +
   SSLCIPH(ANY_TLS12_OR_HIGHER) +
   SSLCAUTH(OPTIONAL) +
   SSLPEER('') REPLACE

   REFRESH SECURITY TYPE(SSL)
   ```

5. Issue the command to show the channel configuration for the channel the application will use

   ```sh
   DISPLAY CHANNEL(DEV.SSL.SVRCONN)
   >  1 : DISPLAY CHANNEL(DEV.SSL.SVRCONN)
    AMQ8414I: Display Channel details.
    CHANNEL(DEV.SSL.SVRCONN)                CHLTYPE(SVRCONN)
    ALTDATE(2024-11-22)                     ALTTIME(16.09.29)
    CERTLABL( )                             COMPHDR(NONE)
    COMPMSG(NONE)                           DESCR( )
    DISCINT(0)                              HBINT(300)
    KAINT(AUTO)                             MAXINST(999999999)
    MAXINSTC(999999999)                     MAXMSGL(4194304)
    MCAUSER( )                              MONCHL(QMGR)
    RCVDATA( )                              RCVEXIT( )
    SCYDATA( )                              SCYEXIT( )
    SENDDATA( )                             SENDEXIT( )
    SHARECNV(10)                            SSLCAUTH(OPTIONAL)
    SSLCIPH(ANY_TLS12_OR_HIGHER)            SSLPEER( )
    TRPTYPE(TCP)
   ```

   The **DEV.SSL.SVRCONN** channel has been configured to use the **ANY_TLS12_OR_HIGHER** CipherSpec. When the **SSLCIPH** option is set, it turns on TLS encryption for any connections to the queue manager using this channel. We will need to specify the same CipherSpec on the client side for the client and server to be able to connect and carry out the TLS handshake.

   The other property to note is **SSLCAUTH**, which is set to **OPTIONAL** in this case. This allows for both 1-Way and 2-Way TLS authentication. The server authentication by the client is mandatory so the server always needs a certificate. This is 1-Way authentication. If the client also has a certificate, 2-way authentication can happen. If a client provides a certificate then it will be used for authentication, however if it does not then client authentication does not happen but the connection is still allowed. We are using 1-Way authentication in this tutorial as only the server has a certificate, so our TLS configuration is set up for encryption and server authentication only. The client authentication is carried out separately using the application name and password.

6. Define the MQ Listener for the SSL PORT and start it

   ```sh
   define listener(LISTENER) trptype(TCP) control(QMGR) port(1419)
   start listener(LISTENER)
   display lsstatus(*) port

   AMQ8631I: Display listener status details.
      LISTENER(SYSTEM.LISTENER.TCP.1)         STATUS(RUNNING)
      PID(255)                                PORT(1414)
   AMQ8631I: Display listener status details.
      LISTENER(LISTENER)                      STATUS(RUNNING)
      PID(428)                                PORT(1419)
   ```

7. End the MQSC interface

   ```sh
   END

   >
   ```

8. Leave the container running and stay in the container command line prompt. It will be needed for the next section.

## Set up client truststore

1. Change to the mounted directory and create a directory for digital certificates

   ```sh
   cd /mnt/mqm
   mkdir -p MQClient/certs
   cd MQClient/certs
   ```

2. Use runmqakm to create a client trustStore

   ```sh
   runmqakm -keydb -create -db client_key.p12 -pw tru5tpassw0rd -type pkcs12 -expire 1000
   ```

3. Check what has been created so far

   ```sh
   ls
    > client_key.p12
   ```

4. Check the contents of the keyStore

   ```sh
   runmqakm -cert -list all -db client_key.p12 -pw tru5tpassw0rd
   > No certificates were found.
   ```

5. Add the public key certificate to the client’s trustStore

   ```sh
   runmqakm -cert -add -label QM1.cert -db client_key.p12 -type pkcs12 -pw tru5tpassw0rd -trust enable -file ../../MQServer/certs/QM1.cert
   ```

6. Check the contents of the keyStore

   ```sh
   runmqakm -cert -list all -db client_key.p12 -pw tru5tpassw0rd
   > Certificates found
   * default, - personal, ! trusted, # secret key
   ! QM1.cert
   ```

7. Exit the Queue Manager container. Be sure to leave the container running.

   ```sh
   exit
   ```

## Expose the client truststore to the sample application

1. From the host command line prompt, copy the client truststore from the queue manager container to the application'sresource directory

   ```sh
   cd ~/mq-examples/openliberty-jarkataee-jms-ibmmq-ssl/src/main/resources/security

   podman cp QM1:/mnt/mqm/MQClient/certs/client_key.p12 client_key.p12
   ```

## Run the application

Install the dependencies and start OpenLiberty:

```sh
mvn install
mvn liberty:dev
```

Once the server is running, send a message to the queue:

```sh
curl -X POST -d 'msg=test message' http://localhost:9080/libertymq/api/enqueue
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
