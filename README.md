# Interactive Brokers: Headless IB Gateway Installation using IBController on Ubuntu Server

This guide was written for anyone that would like to host an instance of the IB Gateway API on an external network. This will allow you to use libraries like [node-ib](https://github.com/pilwon/node-ib) and [ib-sdk](https://github.com/triploc/ib-sdk) in a production environment.

### Influencers
- https://filipmolcik.com/headless-ubuntu-server-for-ib-gatewaytws/
- https://dimon.ca/how-to-setup-ibcontroller-and-tws-on-headless-ubuntu-to-run-two-accounts/
- https://github.com/QuantConnect/Lean/blob/master/DockerfileLeanFoundation
- https://github.com/QuantConnect/Lean/blob/master/Brokerages/InteractiveBrokers/run-ib-controller.sh
- https://github.com/ib-controller/ib-controller/blob/master/userguide.md

## SSH to your server
```shell
# once you've logged in, proceed as `sudo` user
sudo -i
# we'll start in root's home directory
cd ~
```
This is not exactly best Unix practice, but it'll do for now while we get everything up and running.

## Dependencies
```shell
# unzip is used to unzip compressed downloads
apt install unzip
# xvfb is an x11 (GUI) screen simulator
apt install xvfb
# x11vnc is a remote screen simulator viewing tool
apt install x11vnc
```
- `xvfb` will allow IB Gateway to launch because without an x11 container, it crashes.
- `x11vnc` is used to host/serve the simulated x11 GUI allowing you to interact with IB Gateway's user interface remotely.
- [TightVNC](http://www.tightvnc.com/) is an `x11vnc` client you'll need to download to your development computer to remotely access the hosted `x11vnc` instance.
- [tvnjviewer-2.8.3-bin-gnugpl.zip](http://www.tightvnc.com/download/2.8.3/tvnjviewer-2.8.3-bin-gnugpl.zip).

## Setup the x11 screen simulator
```shell
Xvfb :1 -ac -screen 0 1024x768x24 &
# press enter
export DISPLAY=:1
x11vnc -ncache 10 -ncache_cr -display :1 -forever -shared -logappend /var/log/x11vnc.log -bg -noipv6
```
- You'll want to configure your server's firewall using `ufw` or `iptables` to allow port `5900`

## Install IB Gateway
```shell
# in your root's home directory
cd ~
# download installation script
wget https://download2.interactivebrokers.com/installers/ibgateway/latest-standalone/ibgateway-latest-standalone-linux-x64.sh
# make it executable
chmod a+x ibgateway-latest-standalone-linux-x64.sh
# run it
sh ibgateway-latest-standalone-linux-x64.sh -c
```
```shell
# when prompted:
Run IB Gateway?
Yes [y], No [n, Enter]
# choose No
```

## Install IBController
The [IBController](https://github.com/ib-controller/ib-controller) is used to automate the IB Gateway using telnet.
```shell
# still in your root's home directory
cd ~
# get the link to latest IBController from https://github.com/ib-controller/ib-controller/releases
wget https://github.com/ib-controller/ib-controller/releases/download/3.4.0/IBController-3.4.0.zip
unzip ./IBController-3.4.0.zip -d ./ibcontroller.paper
# make the scripts executable
chmod a+x ./ibcontroller.paper/*.sh ./ibcontroller.paper/*/*.sh
```

## Configuration
I'll provide you with my configuration files, but you'll need to modify them for your needs.
#### `~/Jts/jts.ini`
This is the configuration of the IB Gateway itself. You can use your local machine's `jts.ini` as a reference which can be found in the same location path.
```ini
[IBGateway]
WriteDebug=false
TrustedIPs=127.0.0.1
MainWindow.Height=550
RemoteHostOrderRouting=gdc1.ibllc.com
RemotePortOrderRouting=4000
ApiOnly=true
LocalServerPort=4000
MainWindow.Width=700

[Logon]
useRemoteSettings=false
UserNameToDirectory=Ij4tOygzbGV6,djgblkjyvm
Individual=1
tradingMode=p
colorPalletName=dark
Steps=5
Locale=en
SupportsSSL=gdc1.ibllc.com:4000,true,20170813,false
UseSSL=true
s3store=true

[ns]
darykq=1

[Communication]
Internal=false
LocalPort=0
Peer=gdc1.ibllc.com:4001
Region=us
```

#### `~/ibcontroller.paper/IBControllerGatewayStart.sh`
ONLY modify the top part of the file as indicated.
```shell
TWS_MAJOR_VRSN=967
IBC_INI=/root/ibcontroller.paper/IBController.ini
TRADING_MODE=
IBC_PATH=/root/ibcontroller.paper
TWS_PATH=/root/Jts
LOG_PATH=/root/ibcontroller.paper/Logs
TWSUSERID=
TWSPASSWORD=
JAVA_PATH=
```

#### `~/ibcontroller.paper/IBController.ini`
This is the IBController configuration. More info can be found [here](https://github.com/ib-controller/ib-controller/blob/master/userguide.md).
```ini
LogToConsole=no
FIX=no
IbLoginId=<YOUR IB ACCOUNT USERNAME>
IbPassword=<YOUR IB ACCOUNT PASSWORD>
PasswordEncrypted=no
FIXLoginId=
FIXPassword=
FIXPasswordEncrypted=yes
TradingMode=paper
IbDir=
StoreSettingsOnServer=no
MinimizeMainWindow=no
ExistingSessionDetectedAction=manual
AcceptIncomingConnectionAction=accept
ShowAllTrades=no
ForceTwsApiPort=
ReadOnlyLogin=no
AcceptNonBrokerageAccountWarning=yes
IbAutoClosedown=no
ClosedownAt=Saturday 04:11
AllowBlindTrading=yes
DismissPasswordExpiryWarning=no
DismissNSEComplianceNotice=yes
SaveTwsSettingsAt=
IbControllerPort=7462
IbControlFrom=
IbBindAddress=127.0.0.1
CommandPrompt=
SuppressInfoMessages=yes
LogComponents=never
```

## Start the IB Gateway
We'll use the simulated x11 (`DISPLAY=:1`) we created with `xvfb` to start the IBController script.
```shell
DISPLAY=:1 ~/ibcontroller.paper/IBControllerGatewayStart.sh
```

## Firewall
Make sure the following ports are accessible from the outside network:
- `5900`: `x11vnc` remote viewer
- `4002`: default IB Gateway API

## Validate and debug
Use the `TightVNC` to make sure the IB Gateway is up and running.
```shell
# launch the TightVNC app
java -jar tightvnc-jviewer.jar
```

## Caveats
I tried using `nginx` to proxy the websocket connection to port `4002` but was unsuccessful. You'll want to add any external IP's to the `Trusted IPs` list at the bottom of the IB Gateway's `API - Settings`.
![config](http://i.imgur.com/ZhBMjiZ.png)
The IBController should be automatically allowing external IPs, but I was not able to get that working in `v3.4.0`.

## Testing
Using the [ib-sdk](https://github.com/triploc/ib-sdk):
```javascript
const ibsdk = require('ib-sdk')
ibsdk.open({
	clientId: 0,
	host: '123.45.67.890',
	port: 4002,
}, function(error, session) {
	if (error) return console.error('ibsdk.open > error', error);
	let account = session.account()
	console.log('account', account)
})
```

