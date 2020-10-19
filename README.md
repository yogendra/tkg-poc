# TKG PoC

This is a guide to setup a TKG environment with couple of clusters. 

It assumes:

1. TKG 1.2+
1. vSphere
1. Internet Restricted Environment
1. Internet connected jumpbox that will be accessible from TKG cluster

# Download packages
- tkg
- tkg extensions
- crash diagnostics
- OVA files 
- curl
- wget
- mkcert  (Optional) - Required if not internal/corporrate CA certificates given
  ```
  curl -sSL https://github.com/FiloSottile/mkcert/releases/download/v1.4.1/mkcert-v1.4.1-linux-amd64 > $PROJECT_ROOT/bin/mkcert; chmod a+x $PROJECT_ROOT/bin/mkcert
  mkcert -install
  ```  
- octant
  ```
  curl -sSL https://github.com/vmware-tanzu/octant/releases/download/v0.16.1/octant_0.16.1_Linux-64bit.tar.gz  |  tar  --strip-components 1  -C $PROJECT_ROOT/bin -xzvf - octant_0.16.1_Linux-64bit/octant  &&  chmod a+x $PROJECT_ROOT/bin/octant
  ```

# Prepare vSphere Environment
Follow instruction on official documentation to prepare vSphere for TKG installation
- No firewall between Jumpbox to vCenter
- No firewall between cluster (management + workloads) to vCenter
- Network subnet for management and workload cluster
  - DHCP enabled
  - DHCP IP pool should be suficient to run all management and gues cluster VMs (master + worker + extra)
  - Reserved Ips in same subnet for VIPs


# Install TKG Management Cluster

At this point you should go to terminal/shell of you jumpbox and follow the the instructions below. 

## Clone this repo

```
git clone https://github.com/yogendra/tkg-poc ~/tkg-poc
cd ~/tkg-poc
direnv allow
```

## Copy packages

All the downloaded packages, not OVA files, should be copied to ~/tkg-poc/packages. You can use any preferred method(WinSCP/Mac Finder SSH connection/Filezilla)

**TODO/WIP**
1. Automate this with vmw-cli
1. Automate vCenter Tasks 
   1. Use creation 
   2. Permissin assignments
   3. OVA deploy  
   4. template conversion

## Corporate/Private CA

If you are going to use certificates signed by a private CA / corporate CA, copy its certificate to `$PROJECT_ROOT/certs/ca.crt`

## Jumpbox Preparations

1.  Initialize env 
    -  Run init env script

        ```
        01-init-env.sh
        ```
        **OR**
    - Alternatively, you can setup `.env` manually by copying `.env_sample`

      - Create a `.env` file

        ```
        cp .env_sample  .env
        ```

      - Edit `.env` and add following settings

        | Env Var                                     | Description                                                                                       |
        | ------------------------------------------- | ------------------------------------------------------------------------------------------------- |
        | CLIENT                                      | Chort client name. DNS compliant (no spacem, lowercase, no underscore)                            |
        | JUMPBOX_IP                                  | IP address of internet connected VM where images will be pulled and stores                        |
        | POC_DOMAIN                                  | Domain name sufix for POC  (Example: tkg-poc.corp.local)                                          |
        | ROOT_DOMAIN                                 | Internal Root Domain for infrastructure (Example: corp.local)                                     |
        | LOCAL_REGISTRY                              | Local private registry for initialization. This might be on Jumpbox (Example: <jumpbox_ip>:5000 ) |
        | TKG_CUSTOM_IMAGE_REPOSITORY                 | Same as LOCAL_REGISTRY                                                                            |
        | TKG_CUSTOM_IMAGE_REPOSITORY_SKIP_TLS_VERIFY | true if you are using private CA or http only                                                     |
        | GOVC_URL                                    | vCenter URL                                                                                       |
        | GOVC_INSECURE                               | State true if vcenter  is using private CA certs                                                  |
        | GOVC_USERNAME                               | vCenter username (Example: administrator@vsphere.local)                                           |
        | GOCV_PASSWORD                               | vCenter password                                                                                  |
        | VMWUSER                                     | My VMware Username                                                                                |
        | VMWPASS                                     | My VMware Password                                                                                |

1.  Init Jumpbox

    ```
    02-init-jumpbox.sh
    ```

1.  Run registry on jumpbox

    ```
    03-run-registry.sh
    ```

1.  Migrate images

    ```
    04-migrate-tkg-images.sh
    ```


## Prepare TKG Config

Add private/internal CA to the plans

1.  Copy CA certificate and overlay files to`~/.tkg/providers/infrastructure/vsphere/ytt` directory
    ```
    cp $PROJECT_ROOT/overlay/providers/infrastructure-vsphere/ytt/internal-ca.yaml $PROJECT_ROOT/certs/ca.crt ~/.tkg/providers/infrastructure-vsphere/ytt/
    ```

# Setup TKG config using UI
- Run GUI wizard to prepare TKG Config and deploy management cluster
  ```
  ./05-tkg-create-mc.sh
  ```
- Open a browser on Windows/Mac/Linux GUI
- Follow the wizard until all the config steps are done


## Guest Cluster: Shared Services

### Create 

- On the shell run following command to create a guest cluster

  - Provide Guest Cluster Name and Endpoint VIP ip in arguments

  ```
  ./06-tkg-create-guest-cluster.sh poc-shared 10.213.187.198
  ```

### In Cluster Services

- Storage Class  
- Metal LB
- Cert Manager
- Contour
- Harbor
- DEX (Optional)
- ELK (Optional)


## Guest Cluster: Applications

### Create 

- On the shell run following command to create a guest cluster

  - Provide Guest Cluster Name and Endpoint VIP ip in arguments

  ```
  ./06-tkg-create-guest-cluster.sh poc-apps 10.213.187.197
  ```

### In Cluster Services

- Storage Class  
- Metal LB
- Cert Manager
- Contour
- Fluentbit


## Componenet Deployment 

### MetalLB


Create random secret
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)" -o yaml --dry-run=client > metallb/02-secret-emberlist.yaml

