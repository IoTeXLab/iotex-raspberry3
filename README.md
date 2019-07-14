# Tutorial
## IoTeX Blockchain Full Node on Raspberry Pi 3 b+
We will configure our Raspberry Pi 3 in a _headless_ mode, i.e. you will not require any monitor or keyboard, just your PC connected to the same network as the Raspberry. Due to an incompatibility with bolt-DB when compiled with golang for armv6 architecture, we will not install the official Raspbian OS: we need an arm64 OS so we will go with Ubuntu 18.04 Arm64 image for Raspberry 3: 

1. Prepare the micro SD card with the the OS for Raspberry
* Download Ubuntu server 18.04 Arm64 OS for Raspberry Pi 3 from http://cdimage.ubuntu.com/releases/18.04/release/ubuntu-18.04.2-preinstalled-server-arm64+raspi3.img.xz

2. Download Etcher and use it to copy the OS into the micro SD card
* Download Etcher from https://www.balena.io/etcher/
* Insert the Micro-SD into your PC SD reader
* Use Etcher to _flash_ the Ubuntu image file into the SD card

3. First Boot of the Raspberry

* Make sure you have the DHCP enabled in your router/modem
* Connect the Raspberry with an Ethernet cable to your switch/router
* Insert the micr SD into the Raspberry SD slot
* Power on your raspberry (possibly use a 5V power supply that can provide at least 2A)
* Wait a minute for the Raspberry to fully start

6. Find the local IP address of your Raspberry and configure your modem/router
* Open your router config page (usually by typing 192.168.1.1 in a browser)
* Look for _raspberrypi_ in the list of connected devices in your newtork, and take note of the IP address
* Make sure your router will always assign **the same IP** to your Raspberry by configuring a _reserved ip_ for it 
* In my case, the local IP of my Raspberry is 192.168.1.105, so I will use this one but you need to replace it with yours

4. Log in into the Raspberry with SSH

Open a terminal on your PC and type the following (if you are on Windows, you may need to use an [SSH tool like Putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
```
ssh ubuntu@192.168.1.105
use _ubuntu_ as password too
The system may ask you to change your password: just do it and login again
```
5. Update the system (this may take a while, if you are just playing around you can ignore this by now)
```
sudo apt-get update
sudo apt-get upgrade
```
9. Install GoLang 1.12.5 ARM64
```
wget https://dl.google.com/go/go1.12.5.linux-arm64.tar.gz
tar xzf go1.12.5.linux-arm64.tar.gz 
sudo mv go /usr/local
nano .profile
```
Add the following at the end of your .profile file:
```
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$GOPATH/bin:$GOROOT/bin:/home/ubuntu/bin:$PATH
```
Save the file with `Ctrl+X Y [ENTER]` and reload it with:
```
source .profile
```
Check the go version to see it's been installed correctly
```
go version
```
10. Configure some swap space

Building the node with golang will require just a few Mb more than the 1GB provided by the raspberry pi 3 so we will add some swap space for now: that is only needed for the build process, hence we are not going to configure any system setting to have this swap space avaliable on next boots (alternatively, we could just build the node on a different PC and cross-compile for ARM64 architecture - it's not really needed to build it directly on the board!!)
```
sudo fallocate -l 1G /swapfile
sudo dd if=/dev/zero of=/swapfile bs=1024 count=1048576
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

11. Build iotex-core 0.7.2
```
sudo apt-get update
sudo apt-get install git build-essential jq htop
git clone https://github.com/iotexproject/iotex-core.git
cd iotex-core
git checkout v0.7.2
export GO111MODULE=on
make
```
This will take some time: feel free to stop the process with `Ctrl+C` when you see _"go test -short -race ./..."_ as tests may take a long time and they are not required for our purpose.

12. Set the environment to run the node

```
sudo cp ~/iotex-core/bin/ioctl /usr/local/bin
mkdir -p ~/iotex-var
cd ~/iotex-var

export IOTEX_HOME=$PWD

mkdir -p $IOTEX_HOME/data
mkdir -p $IOTEX_HOME/log
mkdir -p $IOTEX_HOME/etc
```
We will start the node on the testnet, as at the time I'm writing the mainnet has not been updated to 0.7.2 yet
```
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/config_testnet.yaml > $IOTEX_HOME/etc/config.yaml
curl https://raw.githubusercontent.com/iotexproject/iotex-bootstrap/master/genesis_testnet.yaml > $IOTEX_HOME/etc/genesis.yaml
curl -L https://t.iotex.me/testnet-data-with-idx-latest > $IOTEX_HOME/data.tar.gz
tar -xzf data.tar.gz
```
13. Configure the node

We will create a wallet account on the testnet, that we will associate to our node:
```
ioctl config set endpoint api.testnet.iotex.one:443
ioctl account createadd operator
```
export the private key of our newely created account
```
ioctl account export operator
```
and list our accounts to see the wallet address of our _operator_ account
```
ioctl account list
```
Now we only need our external (_public_) ip address:
```
wget http://ipinfo.io/ip -qO -
```
and we are ready to edit the config file for our IoTeX node:
```
nano ~/iotex-var/etc/config.yaml
``` 
once the file is open, you need to:
* set your external ip and privkey (look for the placeholder comments in the file). I also like to add a comment with the wallet address corresponding to the private key.
* locate chainDBPath: and set it to /home/ubuntu/iotex-var/data/chain.db
* locate trieDBPath: and set it to /home/ubuntu/iotex-var/data/trie.db
* locate dbPath: and set it to /home/ubuntu/iotex-var/data/poll.db
* locate stderrRedirectFile: and set it to /home/ubuntu/iotex-var/log/s.log

Save and close the file with `Ctrl+X Y [ENTER]`

14. Get ready to start the node

```
mkdir ~/bin
wget https://raw.githubusercontent.com/IoTeXLab/iotex-raspberry3/master/bin/start-node 
chmod +x ~/bin/start-node
start-node | jq
```
The node will take some time to catch up with the latest blockchain status. You can connect to the Raspberry from a different terminal and watch the current status of your node by first pointing the command line client to your local node, then query the node for the infos:

```
ioctl config set endpoint localhost:14014 --insecure
watch ioctl bc info
```
You will get an error message until the node is not fully synced, then you will see the blockchain status: current Epoch, Block Height, tps, etc..
