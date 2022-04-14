# nRF Cloud Provisioning
This repository works as tutorial on how to perform the nRF Cloud Preconnect provisioning process.
You'll learn how to create the required certificates, load them onto the nRF9160 SiP and perform the actual provisioning in a step-by-step guide. You'll also get an understanding of the nRF Cloud API. We will do many steps manually, otherwise use the provided Python script ```device_credentials_installer.py``` to run through this automated.

## Requirements
What you'll need:
1. Python version 3.x.y *(Note: I've used Python 3.7.6)*
2. Download the required Python scripts that we will use from [nRF Cloud utils](https://github.com/nRFCloud/utils)
3. The execution of scripts requires a set Python packages to be installed, see [requirements.txt](https://github.com/nRFCloud/utils/blob/master/python/modem-firmware-1.3%2B/requirements.txt)
4. A nRF Cloud account and your API Key[^1]
5. A nRF9160 SiP with UART / AT command access

## Workspace setup
1. Download the Python scripts from [nRF Cloud utils](https://github.com/nRFCloud/utils)
   - If downloading as zip file, unzip the content of "utils-master.zip\utils-master\python\modem-firmware-1.3+" to a directory of your choice. This directory should contain all Python scripts
2. Open your command prompt that runs Python and change directory to the newly created folder with the downloaded scripts.
3. Install missing Python packages with: ```pip3 install -r requirements.txt```

## Understanding the nRF Cloud Authentication Requirements
Understanding what is needed will make it easier to follow the step-by-step guide. The nRF9160 module requires 3 items in order to communicate with the nRF Cloud API:
1. The AWS Root CA certificate
   - You can simply download the certificate from [AmazonRootCA1.pem](https://www.amazontrust.com/repository/AmazonRootCA1.pem)
2. A private key
   - We will use the new modem functionality (```AT%KEYGEN)``` to securely generate the private key inside of the nRF9160. *Note that you can also create the private key offline, and you will have to if not using modem firmware v1.3.x or higher/newer.*
3. A device certificate for the MQTT connection.
   - You will quickly understand that the majority of steps in this guide is taken in order to generate the device certificate. The device certificate will be created using the Python scripts, it will also be part of your csv file that we upload and therefore commission to nRF cloud. 

## Creating the Certificates, Loading them onto the nRF9160 and Performing the Preconnect Provisioning
### Step 1: Delete any existing keys/certificates on the nRF9160.
You can skip this step if there no certificates/keys stored on the nRF9160 yet.
> Important Hint: The nRF9160 uses a speific certificate storage slot for the nRF Cloud connection. This storage slot, called **security tag**, is identified by its id: **16842753**. Placing the certificates into this slot/security tag will ensure that Nordic proprietary commands (such as AT#XNRFCLOUD from Serial LTE Modem) and Nordic sample applications will use the credentials and run correctly.

a) Ensure the modem is de-registered from the cellular network.
```
AT+CFUN=4
```

b) List the current certifcates/keys on the nRF9160:
```
AT%CMNG=1
```
c) Delete all existing types on sec tag 16842753:
```
AT%CMNG=3,16842753,0
AT%CMNG=3,16842753,1
AT%CMNG=3,16842753,2
```

### Step 2: Generate a private key and retrieve the CSR
The output of the ```AT%KEYGEN``` command is used to create a Certificate Signing Request (CSR). The CSR can be passed to a Certificate Authority (CA) for requesting a client certificate for the device. The modem stores the generated private key to the given security tag (<sec_tag>).

```
AT%KEYGEN=16842753,2,0
```
Example output:
```
AT%KEYGEN=16842753,2,0
%KEYGEN: "MIIBCzCBrwIBADAvMS0wKwYDVQQDDCQ1MDUwMzA0MS0zNTM5LTQ3ZWItODA3Yi0xMDJkYmI0MjNiOWEwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAARhKkti_3EVfSEy7oDyaqqj2cTmlfmhuMdcMOH9xkIX3okdPIFrAZ0sI5xt6WdY56IyQSj8Ce5SxGzMKUuP5ueDoB4wHAYJKoZIhvcNAQkOMQ8wDTALBgNVHQ8EBAMCA-gwDAYIKoZIzj0EAwIFAANJADBGAiEA3AZcIwq61M7PRjdA04z863FmW7mvbgb3lB7YU0bi16QCIQCMbqds9nxdq0XL61QxyuKUgZjPc6Za2-1OUPCVXz9rng.0oRDoQEmoQRBIVhP2dn3hQlQUFAwQTU5R-uAexAtu0I7mkUaAQEAAVgg5g6Cvkt1g0bATSADOVi0euYSuu-7bbBOGYZxgsDyI8tQfQoWN1WvLWECCTaLfZZH1FhAKXmhC0HEeHTZy7qAmgLFx0eRCrFHoHs7UmJ9nsgCgAwWTOEuMH0C6vaQ0OsUT-pDcEWP8tstbFsuvySeOEzHog"
OK
```
Save they Keygen output.

### Step 3: Parse the modem's KEYGEN output and save it to a CSR pem file.
Use the Python script ```modem_credentials_parser.py```.
```
python .\modem_credentials_parser.py -k <your KEYGEN output> -s
```

Example:
```python
python .\modem_credentials_parser.py -k MIIBCzCBrwIBADAvMS0wKwYDVQQDDCQ1MDUwMzA0MS0zNTM5LTQ3ZWItODA3Yi0xMDJkYmI0MjNiOWEwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAARhKkti_3EVfSEy7oDyaqqj2cTmlfmhuMdcMOH9xkIX3okdPIFrAZ0sI5xt6WdY56IyQSj8Ce5SxGzMKUuP5ueDoB4wHAYJKoZIhvcNAQkOMQ8wDTALBgNVHQ8EBAMCA-gwDAYIKoZIzj0EAwIFAANJADBGAiEA3AZcIwq61M7PRjdA04z863FmW7mvbgb3lB7YU0bi16QCIQCMbqds9nxdq0XL61QxyuKUgZjPc6Za2-1OUPCVXz9rng.0oRDoQEmoQRBIVhP2dn3hQlQUFAwQTU5R-uAexAtu0I7mkUaAQEAAVgg5g6Cvkt1g0bATSADOVi0euYSuu-7bbBOGYZxgsDyI8tQfQoWN1WvLWECCTaLfZZH1FhAKXmhC0HEeHTZy7qAmgLFx0eRCrFHoHs7UmJ9nsgCgAwWTOEuMH0C6vaQ0OsUT-pDcEWP8tstbFsuvySeOEzHog -s
```

You should now find two .pem files in the format of *\<nRF9160-UUID\>\_\<sec_tag\>\_\<type\>.pem* in your directory. Example:
```
50503041-3539-47eb-807b-102dbb423b9a_16842753_csr.pem
50503041-3539-47eb-807b-102dbb423b9a_16842753_pub.pem
```

### Step 4: Create a CA Certificate if not owning one already
If you don't already own a CA certificate, you will first need to create the CA certificate in order to create your device certificate(s) afterwards. You can/may use this CA certificate for all your devices, now and in the future. 
You should change the parameters according to your organization and needs. The command will create a self-signed CA certificate, plus a private and public key pair. A self-signed CA certificate is accepted for nRF Cloud. You will need the private key in the following step. Use the Python script ```create_ca_cert.py``` for this step.

Example:
```python
python .\create_ca_cert.py -c GE -st BE -l Berlin -o "Nordic Semiconductor" -ou "Sales" -cn nordicsemi.com -e kevin.kotinkar@nordicsemi.com -f nordic_semi-
```

The output files name format is as follows: 
1. Your CA certificate: ```<your_prefix><CA_serial_number_hex>_ca.pem```
2. The private key associated with the CA certificate: ```<your_prefix><CA_serial_number_hex>_prv.pem``` 
3. The public key associated with the CA certificate: ```<your_prefix><CA_serial_number_hex>_pub.pem```

### Step 5: Create the device certificate
Create the device certificate using the CA certificate, its private key and the CSR file. Use the Python script ```create_device_credentials.py``` for this step. 

Example, giving the device certificate a validity of 2000 days:
```
python .\create_device_credentials.py -ca nordic_semi-0x190194f397b7311508093853b930a2f1354259f3_ca.pem -ca_key nordic_semi-0x190194f397b7311508093853b930a2f1354259f3_prv.pem -csr 50503041-3539-47eb-807b-102dbb423b9a_16842753_csr.pem -dv 2000
```

You should find two .pem files in the format of *\<nRF9160-UUID\>\_\<type\>.pem* in your directory. The file with type ```_crt``` is your newly created device certificate. Example:
```
50503041-3539-47eb-807b-102dbb423b9a_crt.pem
50503041-3539-47eb-807b-102dbb423b9a_pub.pem
```

### Step 6: Download the Amazon Root CA Certificate
Download the [AmazonRootCA1.pem](https://www.amazontrust.com/repository/AmazonRootCA1.pem)

> You should now be set with all three items for the nRF cloud connection. The private key is already stored inside the nRF9160 (see Step 2), means in the following we will only need to upload the Amazon Root CA certificate and the device certificate to the nRF9160.

### Step 7: Upload the certificates to the nRF9160
We will need to upload the Amazon Root CA certificate plus the device certificate to the nRF9160. This is done using the ```AT%CMNG``` command. However, the terminal program you are using must be able to also correctly transmit the new line and carriage return control characters of the certificates.

The command syntax to write the Amazon Root CA certificate to the nRF9160 is:
```
AT%CMNG=0,16842753,0,"<CA_cert_text>"
```
>Important: LTE Link Monitor as terminal program will not correctly format the CR LF endings. Thus, it is required to use the in-built certificate manager when uploading certificates to the nRF9160 using the tool.

[^1]: You find your API key by logging into your [nRF Cloud account](https://nrfcloud.com/#/account), and clicking in the top right corner on "User Account". The Team Details section will show your API Key.
