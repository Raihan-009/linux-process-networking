
# Communicating between process and different namespaces.

This guide outlines the steps to create two namespaces named ***blue-namespace*** and ***lemon-namespace***, and establish a virtual Ethernet network between them using ***veth*** interfaces. The goal is to enable communication between the namespaces and process. Besides allowing them to ping or curl from one namespace to process.

## Prerequisites

- Linux operating system
- Root or sudo access
- Packages

    ```bash
    sudo apt update
    sudo apt upgrade -y
    sudo apt install iproute2
    sudo apt install net-tools
    ```

# Steps

## 1. Enable IP forwarding in the Linux kernel:

   ```shell
   sudo sysctl -w net.ipv4.ip_forward=1
   ```
   This step enables IP forwarding in the Linux kernel, allowing the namespaces to communicate with each other.

## 2. Create namespaces:

![Namespaces](https://blog-bucket.s3.brilliant.com.bd/thumbnail/invalid%20image%20type)

   ```shell
   sudo ip netns add blue-namespace
   sudo ip netns add lemon-namespace
   ```
   This step creates two namespaces named ***blue-namespace*** and ***lemon-namespace***. We can list our namespaces by running `ip netns list`.

   ![namespaces](https://blog-bucket.s3.brilliant.com.bd/thumbnail/ac264478-3c05-4a92-aa41-9e4ab7b0dd2e.png)

## 3. Create the virtual Ethernet link pair:

![Veth Cable](https://blog-bucket.s3.brilliant.com.bd/thumbnail/invalid%20image%20type)

   ```shell
   sudo ip link add veth-blue type veth peer name veth-lemon
   ```
   This command creates a virtual Ethernet link pair consisting of veth-blue and veth-lemon at ***root namespace***.

In order to verify, run `sudo ip link list`

**Expected Output:**

![veth-cable](https://blog-bucket.s3.brilliant.com.bd/thumbnail/0e39dff7-50df-44a8-bd11-80f97b864103.png)

## 4. Set the cable as NIC

![Nic](https://blog-bucket.s3.brilliant.com.bd/thumbnail/invalid%20image%20type)

   ```shell
   sudo ip link set veth-blue netns blue-namespace
   sudo ip link set veth-lemon netns lemon-namespace
   ```
   This command acts as  ***NIC*** link pair consisting of veth-blue and veth-lemon. 

To verify run `sudo ip netns exec blue-namespace ip link` and `sudo ip netns exec lemon-namespace ip link`

**Expected Output**

![nic](https://blog-bucket.s3.brilliant.com.bd/thumbnail/68655634-5fdc-4b75-9385-65de1efcacc1.png)

But as we see, interface has been created but it's **DOWN** and has no ip. Now assign a ip address and turn it **UP**.

## 5. Assign IP Addresses to the Interfaces

![Ip](https://blog-bucket.s3.brilliant.com.bd/thumbnail/invalid%20image%20type)

   ```shell
   sudo ip netns exec blue-namespace ip addr add 192.168.0.1/24 dev veth-blue
   sudo ip netns exec lemon-namespace ip addr add 192.168.0.2/24 dev veth-lemon
   ```
   In this step, IP addresses are assigned to the veth-blue interface in the blue-namespace and to the veth-lemon interface in the lemon-namespace.

To verify run `sudo ip netns exec blue-namespace ip addr` and `sudo ip netns exec lemon-namespace ip addr`

**Expected Output:**
![ip-addr](https://blog-bucket.s3.brilliant.com.bd/thumbnail/85cf7b15-d986-49bb-88f8-89a2d66044fe.png)

## 6. Set the Interfaces Up

![Interface](https://blog-bucket.s3.brilliant.com.bd/thumbnail/invalid%20image%20type)

   ```shell
   sudo ip netns exec blue-namespace ip link set veth-blue up
   sudo ip netns exec lemon-namespace ip link set veth-lemon up
   ```
   These commands set the veth-blue and veth-lemon interfaces ***up***, enabling them to transmit and receive data.

Now run again `sudo ip netns exec blue-namespace ip link` and `sudo ip netns exec lemon-namespace ip link` to verify

**Expected Output:**
![interfaces-up](https://blog-bucket.s3.brilliant.com.bd/thumbnail/4243cbee-3fe8-4405-bf94-6a299e3badf1.png)

## 7. Set Default Routes

   ```shell
   sudo ip netns exec blue-namespace ip route add default via 192.168.0.1 dev veth-blue
   sudo ip netns exec lemon-namespace ip route add default via 192.168.0.2 dev veth-lemon
   ```
   These commands set the **default routes** within each namespace, allowing them to route network traffic.

In order to verify run `sudo ip netns exec blue-namespace ip route` and `sudo ip netns exec lemon-namespace ip route`

**Expected Output:**
![default-route](https://blog-bucket.s3.brilliant.com.bd/thumbnail/99a22088-710a-4355-9d1f-c8bb676dfe13.png)

### In addition, the `route` command in the context of the `ip netns exec` allows you to view the routing table of a specific network namespace. The routing table contains information about how network traffic should be forwarded or delivered.

To view the routing table of the lemon-namespace, we can execute the following command:

```shell
sudo ip netns exec lemon-namespace route
sudo ip netns exec blue-namespace route
```
***Output***
![route](https://blog-bucket.s3.brilliant.com.bd/thumbnail/e35dc728-6d55-4748-a98b-a9165738ecd9.png)

## 8. Test Connectivity

   ```shell
   sudo ip netns exec blue-namespace ping 192.168.0.2
   sudo ip netns exec lemon-namespace ping 192.168.0.1
   ```
   Use these commands to test the connectivity between the namespaces by pinging each other's IP address.

**Expected Output:**
![ping-pong](https://blog-bucket.s3.brilliant.com.bd/thumbnail/379328a1-ba8d-4434-84ba-198af0c85701.png)

## 9. Create a server and run

Now we are ready to create a server inside one namespace and ping or curl from another namespace. Let's create a simple hello-world flask application.

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000, debug=True)
```

Use `sudo nano server.py` and write down these lines. Press `control+O` , `Enter` and `control+X`.

![server.py](https://blog-bucket.s3.brilliant.com.bd/thumbnail/dfefe348-6ba5-4cdd-b81e-bcfe14852d28.png)

To run this server we need to create virtual environment and install packages.

```bash
python3 -m venv venv
source venv/bin/activate
pip3 install flask
```

![flask](https://blog-bucket.s3.brilliant.com.bd/thumbnail/a493c6fa-7393-40c6-b6eb-30e3c901d554.png)


Run `python3 server.py`, but before that, we need to enter the inside of one namespace. Let's try with blue namespace.

```bash
sudo ip netns exec blue-namespace /bin/bash
```

Lets check the ip info. Run `ifconfig`.

![ifconfig](https://blog-bucket.s3.brilliant.com.bd/thumbnail/21f46161-96ce-4c9b-bb6d-c4d7133e241a.png)

Now lets run the application. 
```bash
source venv/bin/activate
python3 server.py
```

![run application](https://blog-bucket.s3.brilliant.com.bd/thumbnail/630172af-d1a7-451a-8c01-7933cbe40589.png)


## 10. Now let's curl from another namespace

Let's get into lemon namespace. Run `sudo ip netns exec lemon-namespace /bin/bash`. Run `ifconfig`

![lemon ns](https://blog-bucket.s3.brilliant.com.bd/thumbnail/bde1479f-e67d-4514-b436-0a0ef2d09624.png)

```bash
curl -v http://192.168.0.1:3000
```

![hello-world](https://blog-bucket.s3.brilliant.com.bd/thumbnail/1946d206-5df6-44ac-8db8-27da3e15642c.png)

Hurray.....!


## 11. Let's create one more server

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello world from process 2'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3001, debug=True)
```


Run it from the blue namespace and curl again from the lemon namespace.

![application2](https://blog-bucket.s3.brilliant.com.bd/thumbnail/407e1336-c34d-4550-a20b-53f33edebb92.png)

```bash
curl -v http://192.168.0.1:3001
```

![process-2](https://blog-bucket.s3.brilliant.com.bd/thumbnail/11951ca7-1358-4eef-9b16-4da09a8d1ece.png)

Holaa...! 

## 11. Clean Up (optional)

   ```shell
   sudo ip netns del blue-namespace
   sudo ip netns del lemon-namespace
   ```
   If you want to remove the namespaces run these commands to clean up the setup.

   # Cheers! 🍻 Have a good day!
