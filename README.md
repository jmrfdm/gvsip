
I'm able to set up a asterisk server with Naf’s gvsip, to work with obi100 and google voice without DDD and OPENPBX. 
It show that asterisk alone could easy work for one google voice number and one sipphone.
It should work with a few google voice numbers and a few sipphones at the same time if you know how to write the extensions.conf manually.  


The tutorial is based on Ubuntu 18.04 minimal (any ubuntu 18.04 will do, minimal will be smallet, you could consider adding ssh server for remote acess ).

##install asterisk
###Clone Naf’s gvsip branch:
```
cd /usr/src
sudo git clone --depth=1 https://github.com/naf419/asterisk.git --branch gvsip  ##narrow clone
cd asterisk
sudo sed -i 's/MAINLINE_BRANCH=.*/MAINLINE_BRANCH=15/' build_tools/make_version
sudo contrib/scripts/install_prereq install
sudo contrib/scripts/get_mp3_source.sh
```

###build asterisk:
```
cd /usr/src/asterisk
sudo ./configure
sudo make menuselect.makeopts

sudo menuselect/menuselect --enable format_mp3 --enable app_macro \
   --enable CORE-SOUNDS-EN-WAV --enable CORE-SOUNDS-EN-ULAW menuselect.makeopts
sudo make
sudo make install
sudo make config
sudo ldconfig
```

###Set initial config:
```
sudo touch /etc/asterisk/{modules,ari,statsd}.conf
sudo cp configs/samples/modules.conf.sample /etc/asterisk/modules.conf #neccsary for load modules
sudo cp configs/samples/logger.conf.sample /etc/asterisk/logger.conf   #for debug, could be deleted after system works  
sudo cp configs/samples/smdi.conf.sample /etc/asterisk/smdi.conf       #for voicemail, could be deleted
```

###Setup rtp.conf
```
cat << EOF | sudo tee /etc/asterisk/rtp.conf
[general]
stunaddr=stun.l.google.com:19302
EOF
```

###set up pjsvp.conf for google voice, could be checked by asteriks command "pjsip show auths"
```
cat << EOF | sudo tee /etc/asterisk/pjsip.conf
[global]
type=global
debug=true
keep_alive_interval=90

; if using chan_sip to host sip clients instead of chan_pjsip,
; you wont have the (required) udp transport that supports those
; clients. if so, just make a dummy one on a port that won't
; conflict with chan_sip
;[incoming-registrations-unused-but-required]
;type=transport
;protocol=udp
;bind=0.0.0.0:9999

[transport_tls]
type=transport
protocol=tls
bind=0.0.0.0:5061


[gvsip1]
type=registration
outbound_auth=gvsip1
server_uri=sip:obihai.sip.google.com
outbound_proxy=sip:obihai.telephony.goog:5061\;transport=tls\;lr\;hide
client_uri=sip:gv17775551234@obihai.sip.google.com
retry_interval=60
support_path=yes
support_outbound=yes
line=yes
endpoint=gvsip1
contact_additional_params=obn=msi
transport=transport_tls
transport_reuse=no

[gvsip1]
type=auth
auth_type=oauth
refresh_token=your_own_refresh_token
oauth_clientid=your_own_oauth_clientid
oauth_secret=your_own_oauth_secret
username=gv17775551234
realm=obihai.sip.google.com

[gvsip1]
type=aor
contact=sip:obihai.sip.google.com

[gvsip1]
type=endpoint
context=from-external
outbound_auth=gvsip1
outbound_proxy=sip:obihai.telephony.goog:5061\;transport=tls\;lr\;hide
aors=gvsip1
direct_media=no
ice_support=yes
rtcp_mux=yes
media_use_received_transport=yes
outbound_registration=gvsip1
EOF
```

###set up sip.conf as sip server, could be checked with asteriks command "sip show peers" and "sip show peer 201".
```
cat <<EOF |sudo tee /etc/asterisk/pjsip.conf
[general]
context=from-internal
bindport=5060

[201]
type=friend
secret = secret1 ;Change to you own password
dtmfmode = rfc2833
host = dynamic
dial=SIP/201
mailbox=201@from-internal
EOF
```

###set up extensions.conf by bridge from-internal and from-external with dialplan, could be checked with asteriks command "dialplan show"
```
cat <<EOF |sudo tee /etc/asterisk/extensions.conf
[from-internal]
; local extensions go here
exten => 201,1,Dial(SIP/201,20)

; These Google Voice outbound calling with no extension prefix
exten => _NXXXXXX,1,Set(CALLERID(dnid)=1xxx${CALLERID(dnid)})   ; replace xxx with your area code
exten => _NXXXXXX,n,Goto(1xxx${EXTEN},1)                        ; replace xxx with your area code
exten => _NXXNXXXXXX,1,Set(CALLERID(dnid)=1${CALLERID(dnid)})
exten => _NXXNXXXXXX,n,Goto(1${EXTEN},1)
exten => _1NXXNXXXXXX,1,Dial(PJSIP/${EXTEN}@gvsip1)

; International call
exten => _011aaXXXXXXXX.,1,Dial(PJSIP/${EXTEN}@gvsip1)          ; replace aa with other national code

[from-external]
exten => s,1,NoOp()
 same => n,Set(crazygooglecid=${CALLERID(name)})
 same => n,Set(stripcrazysuffix=${CUT(crazygooglecid,@,1)})
 same => n,Set(CALLERID(all)=${stripcrazysuffix})
 same => n,Dial(SIP/201,20,D(:1))
EOF
```

###set up your personal information, you have get your google voice number, your own refresh token
   gvnum="yourgoogle voice number"
   token="your own refresh token" #follow instruction from [OAuth 2 refresh_token for Incredible PBX] or [OAuth 2 refresh_token for your own client]
   oauth_clientid=466295438629-prpknsovs0b8gjfcrs0sn04s9hgn8j3d.apps.googleusercontent.com #clientid for [OAuth 2 refresh_token for Incredible PBX] or [OAuth 2 refresh_token for your own client]
   oauth_secret=4ewzJaCx275clcT4i4Hfxqo2 #secret for [OAuth 2 refresh_token for Incredible PBX] or [OAuth 2 refresh_token for your own client]
   sudo sed -i -e 's/gv17775551234/'$gvnum'/' -e 's/your_own_refresh_token/'$token'/' \
               -e 's/your_own_oauth_clientid/'$oauth_clientid'/' -e 's/your_own_oauth_secret/'$oauth_secret'/' /etc/asterisk/pjsip.conf
   
   areacode="your the digit area code"
   natcode="other nation code"
   sudo sed -i -e 's/xxx/'$areacode'/' -e 's/aa/'$natcode'/' /etc/asterisk/sip.conf

###set permissions
   sudo useradd -m asterisk
   sudo chown asterisk. /var/run/asterisk
   sudo chown -R asterisk. /var/{lib,log,spool}/asterisk
   sudo chown -R asterisk. /etc/asterisk /usr/lib/asterisk

##debug asterisk
###start asterisk
  sudo asterisk -cvvvvv

You could see some module load error, that's ok.
Then you'll see  *CLI>, this the place to type in asterisk command
Tips: You could always look up asterisk command after *CLI> by asterisk command "core show help".

### connect to google voice, controlled by pjsip.conf
a few second after *CLI>, your should see both "Transmitting SIP request" and "Received SIP response" message.
If you only see "Transmitting SIP request", and see SSL STATUS_FROM_SSL_ERR, you are using old openssl. 
If a simple update of openssl couldn't fix it, you are out of luck. For an old system , YMMY, I failed to fix the SSL STATUS_FROM_SSL_ERR even upgrade openssl.

### connect to sipphone
It's better to use a softphone from [softphone list] to connect at first and then play with your sipphone. I test my system with MicroSip, it works well.
For phone setting, you need set the ip address of you asterisk machine as server, port is 5060, Transport is UDP, username is 201 and password is secret1.
You could use asterisk command "sip show peers" to check if phone is connected, and use "sip show peer 201" to check the detail of the phone. 

### check the dialplan
Asterisk it's self is a PBX through extensions.conf, which are called dialplan in asterisk.
The dialplan coud be checked by asterisk command "dialplan show". You should see infomation that's very similar to what's in extensions.conf.
Dial other phone number (it's better to not use google number, there's lack of ringback as in [known Issues]) from sipphone to see if the from-internal part works.
Dial your google voice number to see if if the from-external part works. 


##acknowledgement
Thanks for sharing from naf419@github, xekon@freepbx, ward mundy@nerdvittles, llaalways@mitbbs, twinclouds@wordpress

##References
[Nafs Gvsip](https://github.com/naf419/asterisk/tree/gvsip)
[known Issues](https://github.com/naf419/asterisk/wiki)
[pjsip.conf and rtp.conf example](https://github.com/naf419/asterisk/blob/gvsip/README.md)
[asterisk installation] (https://community.freepbx.org/t/how-to-guide-for-google-voice-with-freepbx-14-asterisk-gvsip-ubuntu-18-04/50933)
[OAuth 2 refresh_token for Incredible PBX](http://nerdvittles.com/?p=26204#GVsetup)
[OAuth 2 refresh_token for your own client](http://www.obifirmware.com/OAuth2/)
[softphone list](https://www.mitbbs.com/article_t/CellularPlan/971.html)
[sip.conf and extensions.conf example EN](https://hobbiesbytwinclouds.wordpress.com/2018/05/27/how-to-make-and-receive-calls-using-google-voice-without-xmpp-may-2018-revision/)
[sip.conf and extensions.conf example CN](https://www.mitbbs.com/article_t/PDA/33028435.html)

