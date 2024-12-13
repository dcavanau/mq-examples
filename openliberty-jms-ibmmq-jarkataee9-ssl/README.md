# Open Liberty - JMS & IBM MQ With SSL

A sample application that uses Open Liberty to connect via SSL to IBM MQ to enqueue & dequeue messages. This example uses *1-way authentication*.

The default behavior for the MQ Client application is to ask for the authentication of the MQ queue manager, this is called *1-way authentication*. This is accomplished by specifying **SSLCAUTH(OPTIONAL)** in the definition of the channel.

The method of *2-way authentication* is when additionally, the MQ queue manager asks for the authentication of the MQ Client application. This is accomplished by specifying **SSLCAUTH(REQUIRED)** in the definition of the channel.

Self-signed certificates are used in this example. Self-signed TLS certificates are suitable for personal use or for applications that are used internally within an organization. Self-signed certificates are **NOT** suitable for software used by external users or high-risk or Production environments because the certificates cannot be verified with a Certification
Authority (CA).

## MQ Container Default Configuration

The following users are created:

* User **admin** for administration. This user is created only if the password is set. Secrets must be used to set the password.
* User **app** for messaging (in a group called `mqclient`). This user is created only if the password is set. Secrets must be used to set the password.

Users in `mqclient` group have been given access connect to all queues and topics starting with `DEV.**` and have `put`, `get`, `pub`, `sub`, `browse` and `inq` permissions.

The following queues and topics are created:

* DEV.QUEUE.1
* DEV.QUEUE.2
* DEV.QUEUE.3
* DEV.DEAD.LETTER.QUEUE - configured as the Queue Manager's Dead Letter Queue.
* DEV.BASE.TOPIC - uses a topic string of `dev/`.

Two channels are created, one for administration, the other for normal messaging:

* DEV.ADMIN.SVRCONN - configured to only allow the `admin` user to connect into it. The admin user can be used with the password configured via secret.
* DEV.APP.SVRCONN - does not allow administrative users to connect. Only the `app` user can connect. The password would be as configured by the secret.

## Setup

For this project you'll need to install the following requirements:

* JDK 17 (or later) - E.g. [Temurim](https://adoptium.net/installation/linux)
* Latest Maven, it must be Java 17 compatible - <https://maven.apache.org/install.html>
* Podman
* Jakarta EE 9.1
* OpenLiberty 22.0.0.7
* IBM MQ & Resource Adapter 9.4.1.0
* OpenSSL
* MQ Client Libraries for local machine

## Set up password secrets

Create podman secrets with secret names as “mqAdminPassword” & "mqAppPassword":

* `printf "passw0rd" | podman secret create mqAdminPassword -`
* `printf "passw0rd" | podman secret create mqAppPassword -`

## Set Up MQ Server keystore and certificate

1. Create the self signed certificate on the local machine

   ```sh
   mkdir qm1cert
   cd qm1cert

   openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -keyout QM1.key -out QM1.crt -subj "/C=US/O=IBM/CN=QM1" -nodes
   ls

      QM1.crt QM1.key

   openssl x509 -text -noout -in QM1.crt
   ```

2. Create the keystore to use on the OpenLiberty client (local machine)

   ```sh
   runmqakm -keydb -create -db clientkey.kdb -pw k3ypassw0rd -type pkcs12 -expire 1000 -stash
   ```

3. Import the MQ Server's public key into the client keystore

   ```sh
   runmqakm -cert -add -label QM1.cert -db clientkey.kdb -stashed -trust enable -file QM1.crt

   runmqakm -cert -list all -db clientkey.kdb -stashed

      Certificates found
   * default, - personal, ! trusted, # secret key
   !  QM1.cert
   ```

4. Create a non-running container which creates and mounts a volume onto the container

   ```sh
   podman create --name mq-tls-config-container --mount="type=volume,src=tls-key-vol,dst=/etc/mqm/pki/keys/mykey" --user 0:0 icr.io/ibm-messaging/mq:latest
   ```

5. Copy the QM1 `.key` and `.crt` files to the directory in the container

   ```sh
   podman cp QM1.crt mq-tls-config-container:/etc/mqm/pki/keys/QM1.cert
   podman cp QM1.key mq-tls-config-container:/etc/mqm/pki/keys/QM1.cert
   ```

6. Remove the container

   ```sh
   podman rm mq-tls-config-container
   ```

7. Start the MQ server and verify operation

   ```sh
      podman run -it \
         --name mqtls \
         --secret mqAdminPassword \
         --secret mqAppPassword \
         --env LICENSE=accept \
         --env MQ_QMGR_NAME=QM1 \
         --volume tls-key-vol:/etc/mqm/pki/keys/mykey" \
         --volume qmdata:/mnt/mqm \
         --publish 1414:1414 \
         --publish 8443:9443 \
         --detach \ 
         icr.io/ibm-messaging/mq:latest
   ```

## MQ Queue Manager configuration in container

1. Start the container

   ```sh
   podman run \
     --secret mqAdminPassword \
     --secret mqAppPassword \
     --env LICENSE=accept \
     --env MQ_QMGR_NAME=QM1 \
     --volume qmdata:/mnt/mqm \
     --publish 1414:1414 \
     --publish 1419:1419 \
     --publish 8443:9443 \
     --detach \
     --name QM1 \
     icr.io/ibm-messaging/mq:latest
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
   SSLCIPH('ANY_TLS12_OR_HIGHER') +
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
   define listener(SYSTEM.LISTENER.TCP.2) trptype(TCP) control(QMGR) port(1419)
   start listener(SYSTEM.LISTENER.TCP.2)
   display lsstatus(*) port

   AMQ8631I: Display listener status details.
      LISTENER(SYSTEM.LISTENER.TCP.1)         STATUS(RUNNING)
      PID(255)                                PORT(1414)
   AMQ8631I: Display listener status details.
      LISTENER(LISTENER.TCP.2)                STATUS(RUNNING)
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
   runmqakm -cert -add -label ibmwebspheremqqm1 -db client_key.p12 -type pkcs12 -pw tru5tpassw0rd -trust enable -file ../../MQServer/certs/QM1.cert
   ```

6. Check the contents of the keyStore

   ```sh
   runmqakm -cert -list all -db client_key.p12 -pw tru5tpassw0rd
   > Certificates found
   * default, - personal, ! trusted, # secret key
   ! ibmwebspheremqqm1
   ```

7. Exit the Queue Manager container. Be sure to leave the container running.

   ```sh
   exit
   ```

## Expose the client truststore to the sample application

1. From the host command line prompt, copy the client truststore from the queue manager container to the application'sresource directory

   ```sh
   cd ~/mq-examples/openliberty-jms-ibmmq-jarkataee9-ssl/src/main/liberty/config/resources/security

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
