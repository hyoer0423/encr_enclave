
# AWS Nitro Enclaves Python demo

**WARNING: This project is just proof-of-concept, not production-ready, use at your own risk.**

This project showcase how we can use Python socket package to establish communication between EC2 instance and Nitro Enclave. And use a proxy to make HTTPS call from inside the enclave as usual.

## Architecture

See [Architecture diagram](https://github.com/richardfan1126/nitro-enclave-python-demo/blob/master/docs/architecture.md)

## Installation guide

1. Create an EC2 instance **with Nitro Enclave enabled** in **ap-northeast-1**. See [AWS documentation](https://docs.aws.amazon.com/enclaves/latest/user/create-enclave.html) for steps and requirement.

   Amazon Linux 2 AMI is recommended.

1. Create a new IAM role for the instance, attach `AWSKeyManagementServicePowerUser` policy to the role.

   See [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#working-with-iam-roles) for more detail

1. SSH into the instance. Install `nitro-cli`

   ```
   sudo amazon-linux-extras install aws-nitro-enclaves-cli  
   sudo yum install aws-nitro-enclaves-cli-devel -y  
   sudo usermod -aG ne ec2-user  
   sudo systemctl start nitro-enclaves-allocator.service && sudo systemctl enable nitro-enclaves-allocator.service  
   sudo amazon-linux-extras install docker  
   sudo systemctl start docker  
   sudo systemctl enable docker  
   sudo usermod -a -G docker ec2-user  
   sudo systemctl start docker && sudo systemctl enable docker  
   sudo amazon-linux-extras enable aws-nitro-enclaves-cli  
   sudo yum install -y aws-nitro-enclaves-cli aws-nitro-enclaves-cli-devel   

   ```

1. Modify the preallocated memory for the enclave to 3000 MB.

   Modify the file `/etc/nitro_enclaves/allocator.yaml`, change the following line:

   ```
   # memory_mib: 512
   memory_mib: 3000
   ```

1. Enable Docker and Nitro Enclaves Allocator

   ```
   $ sudo systemctl start nitro-enclaves-allocator.service && sudo systemctl enable nitro-enclaves-allocator.service
   $ sudo systemctl start docker && sudo systemctl enable docker
   ```

1. **Reboot** the instance

1. Loggin to the instance, clone the repository

   ```
   sudo yum install git 
   git clone https://github.com/hyoer0423/encr_enclave.git
   
   ```
 run again 
 ```
   sudo usermod -aG ne ec2-user  
   sudo systemctl start nitro-enclaves-allocator.service && sudo systemctl enable nitro-enclaves-allocator.service  
   sudo systemctl start docker  
   sudo systemctl enable docker  
   sudo usermod -a -G docker ec2-user  
   sudo systemctl start docker && sudo systemctl enable docker  
   sudo amazon-linux-extras enable aws-nitro-enclaves-cli  
```


1. Use the build script to build and run the enclave image

   ```
   $  cd encr_enclave/server
   $ chmod +x build.sh
   $ ./build.sh
   ```


1. After the enclave has launched, you can find the CID of it.

   Find the following line, and take note the `enclave-cid` value   and **PCR0**

```
   Enclave Image successfully created.  
   {
  "Measurements": {
    "HashAlgorithm": "Sha384 { ... }",
    "PCR0": "fdd6b3c0e70ee927046ab974521362a7534a629fdccb195abc69147a133b27b8233ff9153b376af2dccf9503cb43246e",
    "PCR1": "c35e620586e91ed40ca5ce360eedf77ba673719135951e293121cb3931220b00f87b5a15e94e25c01fecd08fc9139342",
    "PCR2": "951c4c27d03d0777288f7de339abdd0640da15d454e0efbe8e29bac74a8e8ea06edda8401b6bb672b1b71d32b9bf6751"
  }
} 
Start allocating memory...
Started enclave with enclave-cid: 16, memory: 2600 MiB, cpu-ids: [1, 17]
{
  "EnclaveID": "i-097a9a35e16a8962c-enc17a99614bf41bf8",
  "ProcessID": 7194,
  "EnclaveCID": 16,
  "NumberOfCPUs": 2,
  "CPUIDs": [
    1,
    17
  ],
  "MemoryMiB": 2600
}
```
 

1. creat [a KMS key](https://ap-northeast-1.console.aws.amazon.com/kms/home?region=ap-northeast-1#/kms/keys)
INSTANCE_ROLE_ARN is the same as the instance of your EC2instance
KMS_ADMINISTRATOR_ROLE allows you to manage the KMS key's Role
PCR0_VALUE_FROM_EIF_BUILD previously recorded PCR0 value
Policy like :
```
   {
    "Version": "2012-10-17",
    "Id": "key-consolepolicy-3",
    "Statement": [
        {
            "Sid": "Enable IAM User Permissions",
            "Effect": "Allow",
            "Principal": {
                "AWS": KMS_ADMINISTRATOR_ROLE
            },
            "Action": "kms:*",
            "Resource": "*"
        },
        {
            "Sid": "Allow access for Key Administrators",
            "Effect": "Allow",
            "Principal": {
                "AWS": INSTANCE_ROLE_ARN
            },
            "Action": [
                "kms:Create*",
                "kms:Describe*",
                "kms:Enable*",
                "kms:List*",
                "kms:Put*",
                "kms:Update*",
                "kms:Revoke*",
                "kms:Disable*",
                "kms:Get*",
                "kms:Delete*",
                "kms:TagResource",
                "kms:UntagResource",
                "kms:ScheduleKeyDeletion",
                "kms:CancelKeyDeletion"
            ],
            "Resource": "*"
        },
        {
            "Sid": "Enable decrypt from enclave",
            "Effect": "Allow",
            "Principal": {
                "AWS": INSTANCE_ROLE_ARN
            },
            "Action": "kms:Decrypt",
            "Resource": "*",
            "Condition": {
                "StringEqualsIgnoreCase": {
                    "kms:RecipientAttestation:ImageSha384": PCR0_VALUE_FROM_EIF_BUILD
                }
            }
        },
        {
            "Sid": "Enable encrypt from instance",
            "Effect": "Allow",
            "Principal": {
                "AWS": INSTANCE_ROLE_ARN
            },
            "Action": "kms:Encrypt",
            "Resource": "*"
        }
    ]
   }
```
 
1. Open a new SSH session, run the `vsock-proxy` tool

   ```
   $ vsock-proxy 8000 kms.ap-northeast-1.amazonaws.com 443
   ```

1. Open another SSH session, install python3 and the necessary packages for running client app

```
   $ yum install python3 -y
   $ cd encr_enclave/client
   $ python3 -m venv venv
   $ source venv/bin/activate
   $ pip install -r requirements.txt
```


1. Run the client app, replace `<cid>` with the enclave CID you get in step 11

   ```
   $ python3 client.py <cid> <key_arn>
   ```
1. output should be like:
```
   {'Plaintext': b'811505', 'Ciphertext': b"\x01\x02\x02\x00x\xae\x83ysZ\xfch%\x1a\x0b\x1d,%`\xec\x7f\x1a\x08>\xcfO\x9f\x98\xcah\xa9\xd9\xacb\xa6\x8e\x8e\x01\xe8xR<9.\xa3\xed\xcb\xd8PX0!W\xa4\x00\x00\x00d0b\x06\t*\x86H\x86\xf7\r\x01\x07\x06\xa0U0S\x02\x01\x000N\x06\t*\x86H\x86\xf7\r\x01\x07\x010\x1e\x06\t`\x86H\x01e\x03\x04\x01.0\x11\x04\x0c\xdc2\x15o\x9c\x0fq\x050\x8eW\xf0\x02\x01\x10\x80!w\xadV\xa6<7O\xf5o\xf3\xd1\x96\xc45\xa0\xf2n~gX'>B#\xf8\xf1o@f.X\x0e\xd6", 'Decryptedtext': '811505'}
```

1. If everthing is OK, you will see the the key ID and key state are shown on the screen.

## File structure

 - `client/`

    This directory contains the code running on parent EC2 instance. Use `requirements.txt` to install necessary pip package
 
 - `server/Dockerfile`

   This file is to build the eif file required to run on the Nitro Enclave.

 - `server/server.py`

   This is the main server app

 - `server/traffic-forwarder.py`

   This is a background app running on Nitro Enclave to forward HTTP traffic between parent instance and enclave via the vsock

 - `server/run.sh`
 
   This is the bootstrap shell script. It first modify the Nitro Enclave netwrok config to assign an IP address `127.0.0.1` to local loopback. Then start the traffic forwarder and the main server app



## Todos

 - Add implementation of attestation
 
License
----

Apache License, Version 2.0