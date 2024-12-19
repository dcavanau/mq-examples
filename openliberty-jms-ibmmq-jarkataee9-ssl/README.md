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

1. Pull and start the IBM MQ docker container:

   ```sh
   podman volume create qmdata
   podman volume create tls-key-vol

   podman pull icr.io/ibm-messaging/mq:latest

   podman run -ti \
     --entrypoint=/bin/bash \
     --volume qmdata:/mnt/mqm \
     --volume tls-key-vol:/etc/mqm/pki/keys/ibmwebspheremqqm1 \
     icr.io/ibm-messaging/mq:latest
   ```

2. Change to the directory that is mounted on the volume to create digital certificates for the server

   ```sh
   cd /mnt/mqm
   mkdir -p MQServer/certs
   mkdir -p MQClient/certs
   cd MQServer/certs
   ```

3. Create the self signed certificate on the server and look at its contents

   ```sh
   openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -keyout QM1.key -out QM1.crt -subj "/C=US/O=IBM/CN=QM1" -nodes
   ls

      QM1.crt QM1.key

   openssl x509 -text -noout -in QM1.crt
   ```

4. Make the server certificate and key are visible to MQ

   ```sh
   cp QM1.* /etc/mqm/pki/keys/ibmwebspheremqqm1/
   ```

5. Create the keystore to use on the OpenLiberty client (local machine)

   ```sh
   runmqakm -keydb -create -db ../../MQClient/certs/client_key.p12 -pw tru5tpassw0rd -type pkcs12 -expire 1000 -stash

   ```

6. Import the MQ Server's public key into the client keystore

   ```sh
   runmqakm -cert -add -label ibmwebspheremqqm1 -db ../../MQClient/certs/client_key.p12 -pw tru5tpassw0rd -trust enable -file QM1.crt

   runmqakm -cert -list all -db ../../MQClient/certs/client_key.p12 -pw tru5tpassw0rd

      Certificates found
   * default, - personal, ! trusted, # secret key
   !  ibmwebspheremqqm1
   ```

7. Exit the container. The certificate data will be safe in the volumes

    ```sh
    exit
    podman container ls --all
    > CONTAINER ID  IMAGE                           COMMAND     CREATED         STATUS                     PORTS                                   NAMES
      1d57f8d01b9b  icr.io/ibm-messaging/mq:latest              24 minutes ago  Exited (0) 27 seconds ago  1414/tcp, 9157/tcp, 9415/tcp, 9443/tcp  great_einstein
    podman container rm 1d57f8d01b9b
    > 1d57f8d01b9b
    ```

## MQ Queue Manager configuration in container

1. Start the MQ server and verify operation

   ```sh
   podman run -it \
   --secret mqAdminPassword \
   --secret mqAppPassword \
   --env LICENSE=accept \
   --env MQ_QMGR_NAME=QM1 \
   --volume tls-key-vol:/etc/mqm/pki/keys/ibmwebspheremqqm1 \
   --volume qmdata:/mnt/mqm \
   --publish 1414:1414 \
   --publish 8443:9443 \
   --detach \
   --name QM1 \
   icr.io/ibm-messaging/mq:latest

   podman logs QM1
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

4. Verify security has been enabled

   ```sh
   DISPLAY CHANNEL(DEV.APP.SVRCONN)
        1 : DISPLAY CHANNEL(DEV.APP.SVRCONN)
   AMQ8414I: Display Channel details.
   CHANNEL(DEV.APP.SVRCONN)                CHLTYPE(SVRCONN)
   ALTDATE(2024-12-18)                     ALTTIME(16.21.08)
   CERTLABL( )                             COMPHDR(NONE)
   COMPMSG(NONE)                           DESCR( )
   DISCINT(0)                              HBINT(300)
   KAINT(AUTO)                             MAXINST(999999999)
   MAXINSTC(999999999)                     MAXMSGL(4194304)
   MCAUSER(app)                            MONCHL(QMGR)
   RCVDATA( )                              RCVEXIT( )
   SCYDATA( )                              SCYEXIT( )
   SENDDATA( )                             SENDEXIT( )
   SHARECNV(10)                            SSLCAUTH(OPTIONAL)
   SSLCIPH(ANY_TLS12_OR_HIGHER)            SSLPEER( )
   TRPTYPE(TCP)
   ```

   The **DEV.SSL.SVRCONN** channel has been configured to use the **ANY_TLS12_OR_HIGHER** CipherSpec. When the **SSLCIPH** option is set, it turns on TLS encryption for any connections to the queue manager using this channel. We will need to specify the same CipherSpec on the client side for the client and server to be able to connect and carry out the TLS handshake.

   The other property to note is **SSLCAUTH**, which is set to **OPTIONAL** in this case. This allows for both 1-Way and 2-Way TLS authentication. The server authentication by the client is mandatory so the server always needs a certificate. This is 1-Way authentication. If the client also has a certificate, 2-way authentication can happen. If a client provides a certificate then it will be used for authentication, however if it does not then client authentication does not happen but the connection is still allowed. We are using 1-Way authentication in this tutorial as only the server has a certificate, so our TLS configuration is set up for encryption and server authentication only. The client authentication is carried out separately using the application name and password.

5. End the MQSC interface

   ```sh
   END

   >
   ```

6. Exit the Queue Manager container. Be sure to leave the container running.

7. Create a test directory and pull the client keystore and stash files

   ```sh
   mkdir mqcerts
   cd mqcerts

   podman cp QM1:/mnt/mqm/MQClient/certs/client_key.p12 client_key.p12
   podman cp QM1:/mnt/mqm/MQClient/certs/client_key.sth client_key.sth
   ls
   ```

8. Use `amqssslc` to test the connection. You will be prompted for the `app` user's password.

   ```sh
   export MQSAMP_USER_ID=app

   amqssslc -m QM1 -c DEV.APP.SVRCONN -x 'localhost(1414)' -k "/Users/davidcavanaugh/mqcerts/client_key.p12" -s ANY_TLS12_OR_HIGHER -l ibmwebspheremqqm1

   Sample AMQSSSLC start
   Connecting to queue manager QM1
   Using the server connection channel DEV.APP.SVRCONN
   on connection name localhost(1414).
   Using SSL CipherSpec ANY_TLS12_OR_HIGHER
   Using SSL key repository stem /Users/davidcavanaugh/mqcerts/client_key.p12
   Certificate Label: ibmwebspheremqqm1
   No OCSP configuration specified.
   Enter password: ********
   Connection established
   ```

   This shows that a client can connect to the MQ Server using SSL/TLS security.

## Expose the client truststore to the sample application

1. From the host command line prompt, copy the client truststore from the queue manager container to the application'sresource directory

   ```sh
   cd ~/mq-examples/openliberty-jms-ibmmq-jarkataee9-ssl/src/main/liberty/config/resources/security

   podman cp QM1:/mnt/mqm/MQClient/certs/client_key.p12 client_key.p12
   podman cp QM1:/mnt/mqm/MQClient/certs/client_key.sth client_key.sth
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
