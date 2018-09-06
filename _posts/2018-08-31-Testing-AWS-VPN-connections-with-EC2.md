---
layout: post
title: Testing AWS VPN connections with EC2
banner: /assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/banner.png
author: lvandonkersgoed

---
Many of our customers use VPNs to set up secure connections to their AWS environments. A few common use cases for VPNs are hybrid clouds, remote backups, and federated user management. This article will describe how to test VPN connections without requiring access to the remote end.

Let's take a pretty straightforward VPN setup, where an on-premise database replicates its contents to an RDS instance:
![Simple VPN](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/simple-vpn.png)

In this scenario, you are responsible for setting up the AWS side, and you're working with the IT department that manages the corporate data center. The remote engineer indicates that he can't connect to the RDS instance, but there is no way for you to log in and check their configuration for them. In this case, simulating the connection might help you determine if the problem is located on the AWS side or in their data center.

To simulate their VPN connection, we will setup a new EC2 instance functioning as a Customer Gateway. We will use Openswan to set up a site-to-site VPN connection with the existing VPC VPN. The instance will be placed in a separate VPC to guarantee complete isolation of the other AWS resources.
![EC2 CGW](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/ec2-cgw.png)

# Starting point
At the start of this tutorial, we assume that you already have an existing VPC with a database in a private subnet, as depicted in the top image. Of course you are free to replace the RDS instance with a Redis Cluster, EC2 instance, or any other resource that is only accessible from within the VPC. A Virtual Private Gateway, Customer Gateway and existing VPN connection can be present, but it is not required.

In the rest of this tutorial, we will refer to the existing VPC as the "RDS VPC".

# Setup a new VPC
The first thing to do is to setup a simple VPC. Navigate to the VPC section in the AWS Web Console and click "Create VPC". Give the VPC a CIDR block that does not overlap your RDS VPC CIDR block. In my case, the RDS VPC has a `10.0.0.0/16` CIDR, so I'm assigning a `192.168.0.0/16` CIDR to the VPN VPC.

![Create VPC](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/create-vpc.png)

## Create a new subnet
Next, create a subnet to host our VPN instance. You can give it the same CIDR as your VPC, or a smaller one. 

![Create Subnet](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/create-subnet.png)

## Create a new internet gateway
Then create an internet gateway. This only requires a name.

![Create IGW](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/create-igw.png)

The next step is to assign the internet gateway to your VPC.

![Attach IGW](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/attach-igw.png)

## Update the route table
The last thing required to setup our VPC is to edit the route table and make sure all traffic is routed over the internet gateway.

![Update Route Table](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/update-route-table.png)

# Setup the EC2 instance
Create a new EC2 instance running Amazon Linux and place it in the newly created subnet. Make sure it has a public IP address. I used `Amazon Linux AMI 2018.03.0 (HVM)` running on a `t2.micro`.

![New EC2 Instance](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/new-ec2-instance.png)

# Configure VPN and download its configuration
Now that we have an instance that will function as our Customer Gateway, we can configure it in our RDS VPC. Switch back to that VPC and navigate to the "Customer Gateways" section. Click "Create Customer Gateway" and fill in a name and the public IP address of the EC2 instance you have just created.

![New Customer Gateway](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/create-customer-gateway.png)

If you have not done so before, setup a Virtual Private Gateway and attach it to your RDS VPC.

![New Virtual Private Gateway](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/create-vpg.png)

Now navigate to "VPN Connections" section and click "Create VPN Connection". Select the Virtual Private Gateway and Customer Gateway you have created in the previous steps. For Routing Options, choose "Static" and fill in the CIDR of your other VPC (`192.168.0.0/16` in our case).

![New VPN](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/create-vpn.png)

Click "Create VPN Connection". The last thing to do is to update the route tables and security groups, so that the RDS database knows how to communicate with our new Customer Gateway. Navigate to the route table for the subnet(s) that contain your RDS database and add the following rule. This instructs the router to route any traffic destined for `192.168.0.0/16` over the Virtual Private Gateway.

![Add Route](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/add-route.png)

The final step is to update the security group for RDS and allow incoming traffic from the remote subnet. Open the security group for RDS and add this rule:

![Add Security Group Rule](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/add-sg-rule.png)

Save the security group rules and you're done configuring the RDS VPC. Navigate to the "VPN Connections" section one last time, select your newly created VPN connection and click "Download Configuration". Select Openswan, and click the Download button.

![Download config](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/download-config.png)

# Configure the EC2 instance
SSH into the instance and download openswan 2.6.37:
```
wget packages.us-west-2.amazonaws.com/2013.09/main/131d92c211f1/x86_64/Packages/openswan-2.6.37-2.16.amzn1.x86_64.rpm
```

Then install it:
```
sudo yum install ./openswan-2.6.37-2.16.amzn1.x86_64.rpm
```

Now open the configuration file you've downloaded in the previous chapter and follow its instructions. The first step is to open `/etc/sysctl.conf` and add a few lines. In my experience, the official instructions are missing two lines, so the content I used is this:
```
net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.send_redirects=0
net.ipv4.conf.default.accept_redirects=0
```

Close the file and apply with `sysctl -p`. Then open `/etc/ipsec.conf` and make sure that the line `include /etc/ipsec.d/*.conf` is not commented out.

Now open `/etc/ipsec.d/aws.conf` and paste and update the contents as described in the configuration file. In my case that would look like this:

```
conn Tunnel1
	authby=secret
	auto=start
	left=%defaultroute
	leftid=34.240.4.56
	right=52.48.221.159
	type=tunnel
	ikelifetime=8h
	keylife=1h
	phase2alg=aes128-sha1;modp1024
	ike=aes128-sha1;modp1024
	auth=esp
	keyingtries=%forever
	keyexchange=ike
	leftsubnet=192.168.0.0/16
	rightsubnet=10.0.0.0/16
	dpddelay=10
	dpdtimeout=30
	dpdaction=restart_by_peer
```

Follow the instructions for creating the `/etc/ipsec.d/aws.secrets` file.

The configuration document also contains instructions for a second tunnel, but we don't need that, so you can skip it.

Execute `sudo service ipsec restart` to apply your changes. Then run `sudo service ipsec status` and you should see something like this:
```
[ec2-user@ip-192-168-0-229 ~]$ sudo service ipsec status
IPsec running  - pluto pid: 3876
pluto pid 3876
1 tunnels up
some eroutes exist
```

Then run a port test, and if all is well you should now be able to connect to the RDS instance!
```
[ec2-user@ip-192-168-0-229 ~]$ nc -zv mydb.caokygpizz5t.eu-west-1.rds.amazonaws.com 3306
Connection to mydb.caokygpizz5t.eu-west-1.rds.amazonaws.com 3306 port [tcp/mysql] succeeded!
```

# Troubleshooting
If for any reason things are not working as they should, check the following things:

## Is the tunnel up?
In the RDS VPC, go to "VPN Connections", select your testing VPN and click the "Tunnels" tab. It should show one tunnel up:

![Tunnels](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/tunnels.png)

Also double check if the route tables for the subnet containing your private resource (RDS in our case) has a route to the Virtual Private Gateway:

![Add Route](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/add-route.png)

The last common mistake is a security group that does not allow inbound access from the external subnet. So double check that too:

![Add Security Group Rule](/assets/posts/2018-08-31-Testing-AWS-VPN-connections-with-EC2/add-sg-rule.png)

# A final note
The scenario above might seem silly to you. Why would you setup a VPN connection to check if you're able to connect to resources in the VPC? It might be a lot easier to just spin up an instance in a public subnet and check from there. Also, verifying that things work from your new VPN will not always help in determining the cause of issues you have with the other VPN.

There are a few good reasons to still setup your own VPN though:
1. If things work from your new VPN, you can at least guarantee that what you are trying to achieve is generally possible.
2. Going through the motions of updating route tables and security groups might lead you to a configuration setting you missed in the original VPN.
3. There are a few things that do work from within a VPC, but not over VPN. An example is resolving DNS names of VPC Interface Endpoints (see the [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpce-interface.html#vpce-interface-limitations)). Although the DNS names do not resolve, the interfaces _are_ reachable over VPN. Testing and confirming this was my actual motivation for setting up a testing VPN, leading to this blog post.
4. Setting up an EC2 based VPN allows AWS engineers to experience working with VPN connections without requiring access to the expensive hardware routers and firewalls commonly connected to AWS VPNs, such as Cisco ASA, Juniper SRX or Palo Alto PA.

I am sure there are other reasons for using EC2 instances to set up VPN connections I haven't thought of. I hope this post will help anyone achieve this setup in the future.

