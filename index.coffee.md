This is a complete, runnable example for `useful-wind` with a Docker.io FreeSwitch image.
The Docker image will run on your local interface (Docker's `--net host`) so you should be able to test with a standard SIP client.

Usage
-----

Assuming you have recent Node.js and Docker.io installed:

```shell
./run.sh
```

Application
-----------

The script doesn't do much besides issuing `answer` and then `sleep` commands, and logging DTMF.

    CallServer = require './node_modules/useful-wind/call_server.coffee.md'
    isset = require 'isset'
    dateFormat = require 'dateformat'
    eventHandler =''
    globalObj =''
    ivrBase ='/usr/local/freeswitch/sounds/en/us/callie'
    arrEvents =['ExecuteWelcome','DidiValidate','ValidateDTMF','ConfLocked','NoConfDate','BalanceFinished','NoCreditInAccount','AccountBlocked','AdminNotCome','PinAccepted','InvalidPin','ParticipantAdded']
    arrHangupEvent =['NoConfDate', 'BalanceFinished', 'NoCreditInAccount', 'AccountBlocked', 'ConfLocked']

    test =
      include: (ctx) ->

 When PHONE connect is answring

        @action 'answer'
        .then ->
        eventHandler = @action
        globalObj = this
        #console.log "Session ==== #{JSON.stringify(ctx.data, null, "    ")}"
        #console.log "Caller-Unique-ID ==== #{this.data['Caller-Unique-ID']}"
        #console.log "Unique-ID ==== #{this.data['Unique-ID']}"
        this.call.data[this.data['Unique-ID']] = {
          ExecuteWelcome:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},ExecuteWelcome}"},
          DidiValidate:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},DidiValidate}"},
          ValidateDTMF:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},ValidateDTMF}"},
          ConfLocked:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},ConfLocked}"},
          NoConfDate:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},NoConfDate}"},
          BalanceFinished:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},BalanceFinished}"},
          NoCreditInAccount:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},NoCreditInAccount}"},
          AccountBlocked:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},AccountBlocked}"},
          AdminNotCome:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},AdminNotCome}"},
          PinAccepted:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},PinAccepted}"},
          InvalidPin:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},InvalidPin}"},
          ConfrenceStablished:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},ConfrenceStablished}"},
          ParticipantAdded:{appUUID:'', appData:"{uniqueID=#{this.data['Unique-ID']},ParticipantAdded}"},
          DidRequest:{},
          DidResponse:{},
          DTMF: [],
          DTMFRequest:{},
          DTMFResponse:{},
          DTMFValidationComplete:'',
          PINValidationComplete:'',
          PINAcceptedIVRComplete:'',
          Conference:'No',
          AcParticipantRequest:{},
          JoinParticipantId:0,
          RecordingFilePath:'',
          UserType:''
        }
        jsonObj = {
            callerUser : this.data['Caller-Username'],
            DID : this.data['Caller-Destination-Number'],
            reqType: "validateDID",
            uniqueID: this.data['Unique-ID'],
            module: "AC"
          }
        this.data[this.data['Unique-ID']].DidRequest =jsonObj
        eventHandler 'playback', "{uniqueID=#{this.data['Unique-ID']},ExecuteWelcome}#{ivrBase}/ivr/8000/ivr-welcome_to_freeswitch.wav"

DTMF Event

        @call.on 'DTMF', (res) ->
              digit = res.body['DTMF-Digit']
              console.log "Received #{digit}"
              this.data[this.data['Unique-ID']].DTMF.push(digit)

Validate DTMF

              if digit is '#'
                  arrDTMF = this.data[this.data['Unique-ID']].DTMF
                  index = arrDTMF.indexOf("#")
                  if index > -1
                       arrDTMF.splice(index, 1)
                  final_pin = arrDTMF.join("")
                  console.log "Final PIN === #{final_pin}"
                  this.data[this.data['Unique-ID']].DTMF =[]
                  jsonObj = {
                      callerUser : this.data['Caller-Username'],
                      DID : this.data['Caller-Destination-Number'],
                      reqType: "validatePIN",
                      PIN: final_pin,
                      uniqueID: this.data['Unique-ID'],
                      module: "AC"
                    }
                  this.data[this.data['Unique-ID']].DTMFRequest =jsonObj
                  eventHandler 'set', "{uniqueID=#{this.data['Unique-ID']},ValidateDTMF}"

Message Event from fsw

        @call.on 'CHANNEL_EXECUTE_COMPLETE', (res) ->
              respBody = res.body
              #console.log "CHANNEL_EXECUTE_COMPLETE #{JSON.stringify(res, null, "    ")}"
              appData = if isset(respBody['Application-Data']) then respBody['Application-Data'] else ''
              appUUID =respBody['Application-UUID']
              arrUserInfo = this.data[this.data['Unique-ID']]
              #console.log "app Data  #{appData}"
              if appData
                 arrAppData =appData.split('/')
                 if arrAppData[0]
                   arrAppDataCheck =arrAppData[0].split(',')
                   if arrAppDataCheck[1]
                      arrAppDataCheck =arrAppDataCheck[1].replace(/}\s*$/, "")
                      #console.log "CHANNEL EXECUTE COMPLETE == #{arrAppDataCheck}"
                      if arrAppDataCheck && arrAppDataCheck in arrEvents
                          if arrAppDataCheck && appUUID is arrUserInfo[arrAppDataCheck].appUUID
                              arrUserInfo[arrAppDataCheck].appUUID =''
                              if arrAppDataCheck is 'ExecuteWelcome'
                                    socket.emit 'validateDID', arrUserInfo.DidRequest
                                    console.log "Welcome ivr completed."
                                    #console.log "Arr Data === #{JSON.stringify(arrUserInfo.DidRequest, null, "    ")}"
                              else if arrAppDataCheck is 'DidiValidate'
                                    console.log "Phone is hangup due to DID is not a valid."
                                    eventHandler 'hangup'
                              else if arrAppDataCheck is 'ValidateDTMF'
                                    console.log "Validate DTMF"
                                    socket.emit 'validatePIN', arrUserInfo.DTMFRequest
                                    arrUserInfo.DTMFRequest={}
                                    arrUserInfo.DTMFValidationComplete='Yes'
                              else if arrAppDataCheck is 'InvalidPin'
                                    console.log "Invalid PIN, Try again!"
                                    arrUserInfo.DTMFRequest={}
                                    arrUserInfo.DTMFValidationComplete='No'
                                    eventHandler "play_and_get_digits","1 11 3 5000 # #{ivrBase}/conference/8000/conf-pin.wav", this.data['Unique-ID']
                              else if arrAppDataCheck is 'PinAccepted'
                                  console.log "PIN is accepted."
                                  arrUserInfo.DTMFRequest={}
                                  arrUserInfo.DTMFValidationComplete=''
                                  arrUserInfo.PINAcceptedIVRComplete=''

Stat confrence code here this is executed when PIN is accepted and after played IVT PIN accepted

                                  arrResponse = arrUserInfo.DTMFResponse
                                  now = new Date()
                                  curentDate = dateFormat(now, "yyyy-mm-dd HH:MM:ss")
                                  checkconfStatus =1
                                  console.log "After starttime = #{arrResponse.ac_conf.starttime} ,After endtime = #{arrResponse.ac_conf.endtime} and servertime= #{curentDate}"
                                  if isset(arrResponse.ac_conf.starttime) && isset(arrResponse.ac_conf.endtime)
                                       console.log "Starttime and endtime is set"
                                       if arrResponse.ac_conf.endtime >= curentDate && arrResponse.ac_conf.starttime <= curentDate
                                            console.log "Date and Time is okay for conf of Reserver one"
                                       else
                                            console.log "Date and Time is NOT okay for conf of Reserver one"
                                            checkconfStatus =0
                                            eventHandler 'playback', "{uniqueID=#{this.data['Unique-ID']},NoConfDate}#{ivrBase}/NoConfDate.wav"
                                  else
                                      console.log "Unreserve Conference so No Date and Time needs"
                                  if arrResponse.ac_conf.confpop is 0 && checkconfStatus is 1
                                       arrResponse.defaultConfPop=arrResponse.ac_cust_info.defaultpop
                                       console.log "Conf POP ID= #{arrResponse.defaultConfPop}"
                                  if checkconfStatus is 1
                                       if arrResponse.cc_card.status is 1
                                           if arrResponse.cc_card.typepaid is 1
                                                bal = arrResponse.cc_card.creditlimit + arrResponse.cc_card.credit
                                                console.log "Balance= #{bal}"
                                                if bal < 0
                                                   console.log "No Balance in your account"
                                                   checkconfStatus =0
                                                   eventHandler 'playback',"{uniqueID=#{this.data['Unique-ID']},BalanceFinished}#{ivrBase}/balanceFinished.wav"
                                           else
                                              if arrResponse.cc_card.credit<=0
                                                  console.log "No credit amount in your account"
                                                  checkconfStatus =0
                                                  #checkconfStatus =1
                                                  eventHandler 'playback',"{uniqueID=#{this.data['Unique-ID']},NoCreditInAccount}#{ivrBase}/balanceFinished.wav"
                                       else
                                           checkconfStatus =0
                                           eventHandler 'playback',"{uniqueID=#{this.data['Unique-ID']},AccountBlocked}#{ivrBase}/AccountBlock.wav"
                                       if checkconfStatus is 1
                                           if arrResponse.ac_conf.locked is 1
                                               console.log "conference is locked"
                                               eventHandler "playback","{uniqueID=#{this.data['Unique-ID']},ConfLocked}#{ivrBase}/conference/8000/conf-locked.wav"
                                           else
                                               arrParticipant ={
                                                 reqType:'addParticipant',
                                                 module: 'AC',
                                                 UniqueID: this.data['Unique-ID'],
                                                 arrData:{
                                                     uniqueid: this.data['Caller-Unique-ID'],
                                                     starttime:dateFormat(now, "yyyy-mm-dd HH:MM:ss"),
                                                     endtime:dateFormat(now, "yyyy-mm-dd HH:MM:ss"),
                                                     duration:0,
                                                     callstatus:1,
                                                     callerid:this.data['Caller-Username'],
                                                     calleridname:this.data['Caller-Username'],
                                                     callerchannel:this.data['Caller-Unique-ID'],
                                                     calleruniqueid:this.data['Caller-Unique-ID'],
                                                     id_ac_conf:arrResponse.ac_conf.id,
                                                     confmanager:0,
                                                     callername:this.data['Caller-Username'],
                                                     file:'',
                                                     initial_user:0
                                                 }
                                               }
                                               arrUserInfo.AcParticipantRequest = arrParticipant
                                               if arrResponse.PIN is arrResponse.ac_conf.userpin
                                                   arrUserInfo.UserType ="Participiant"
                                                   feature ='unmute,'
                                                   if arrResponse.ac_conf.moh is 1
                                                       feature ="nomoh,"
                                                   arrResponse.feature =feature
                                                   socket.emit 'addParticipant', arrUserInfo.AcParticipantRequest
                                               else if arrResponse.ac_conf.adminpin is arrResponse.PIN
                                                   arrUserInfo['AcParticipantRequest'].arrData.confmanager =1
                                                   arrUserInfo.UserType ="Admin"
                                                   arrResponse.feature ="unmute,moderator"
                                                   socket.emit 'addParticipant', arrUserInfo.AcParticipantRequest
                              else if arrAppDataCheck is 'AdminNotCome'
                                   console.log "Admin Not come"
                                   eventHandler "set","conference_flags=wait-mod,audio-always"
                                   eventHandler "conference","#{arrUserInfo.DTMFResponse.ac_conf.id}\@default++flags{#{arrUserInfo.DTMFResponse.feature}}"
                              else if arrAppDataCheck is 'ParticipantAdded'
                                  arrDTMFResponse = arrUserInfo.DTMFResponse
                                  #console.log "File path ==#{arrUserInfo.RecordingFilePath},   #{arrDTMFResponse.ac_conf.recording}"
                                  if arrUserInfo.RecordingFilePath && arrDTMFResponse.ac_conf.recording is 1
                                       eventHandler "set","conference_auto_record=#{arrUserInfo.RecordingFilePath}"
                                  if arrUserInfo.UserType is 'Participiant'
                                       adminNotcome =0
                                       if arrDTMFResponse.ac_conf.startafteradmin is 1 && arrDTMFResponse.ac_conf.adminjoin is 0
                                              console.log "start after admin join"
                                              adminNotcome =1
                                              eventHandler "playback","{uniqueID=#{this.data['Unique-ID']},AdminNotCome}#{ivrBase}/AdminNotcome.wav"
                                       else if arrDTMFResponse.ac_conf.startafteradmin is 1 && arrDTMFResponse.ac_conf.adminjoin is 1
                                              console.log "both are true, Do something"
                                       if adminNotcome is 0
                                           eventHandler "conference","#{arrDTMFResponse.ac_conf.id}\@default++flags{#{arrDTMFResponse.feature}}"
                                           console.log "Conference started by #{arrUserInfo.UserType}"
                                  else if arrUserInfo.UserType is 'Admin'
                                       eventHandler "conference","#{arrDTMFResponse.ac_conf.id}\@default++flags{#{arrDTMFResponse.feature}}"
                                       console.log "Conference started by #{arrUserInfo.UserType}"
                              else if arrAppDataCheck in arrHangupEvent
                                   console.log "Phone is hangup"
                                   eventHandler 'hangup'
                              else
                                  console.log "Default event is fired ==#{arrAppDataCheck}"

        @call.on 'CHANNEL_EXECUTE', (res) ->
              respBody =res.body
              appData = if isset(respBody['Application-Data']) then respBody['Application-Data'] else ''
              appUUID =respBody['Application-UUID']
              arrUserInfo = this.data[this.data['Unique-ID']]
              #console.log "App Data ==#{arrUserInfo}";
              if appData
                   arrAppData =appData.split('/')
                   if arrAppData[0]
                       arrAppDataCheck =arrAppData[0].split(',')
                       if arrAppDataCheck[1]
                            arrAppDataCheck =arrAppDataCheck[1].replace(/}\s*$/, "")
                            if arrAppDataCheck && arrAppDataCheck in arrEvents
                                 arrUserInfo[arrAppDataCheck].appUUID = appUUID

 CHANNEL_HANGUP Event

        @call.on 'CHANNEL_HANGUP', (res) ->
           channel_data = res.body
           console.log "Call has been cancelled  Unique-ID ==== #{channel_data['Caller-Unique-ID']}"
           now = new Date()
           curentDate = dateFormat(now, "yyyy-mm-dd HH:MM:ss")
           jsonObj ={
             reqType:'updateParticipant',
             module: 'AC',
             UniqueID: channel_data['Unique-ID'],
             condition:{
                 uniqueid: channel_data['Unique-ID']
               }
            setValue:{
                callstatus:0,
                endtime:curentDate,
                initial_user:0
            }
           }
           socket.emit 'updateParticipant', jsonObj

 CHANNEL_HANGUP_COMPLETE Event

        @call.on 'CHANNEL_HANGUP_COMPLETE', (res) ->
           channel_data = res.body
           console.log "Call has been complete  Caller-Unique-ID ==== #{channel_data['Caller-Unique-ID']}"
           now = new Date()
           curentDate = dateFormat(now, "yyyy-mm-dd HH:MM:ss")
           jsonObj ={
             reqType:'updateParticipant',
             module: 'AC',
             UniqueID: channel_data['Unique-ID'],
             condition:{
                 uniqueid: channel_data['Caller-Unique-ID']
               }
            setValue:{
                callstatus:0,
                endtime:curentDate,
                initial_user:0
            }
           }
           socket.emit 'updateParticipant', jsonObj

Configuration. In this case this only lists the middleware(s) we'll be using.

    cfg =
      use: [
        test
      ]

    CallServerObj = new CallServer cfg
    CallServerObj.listen 7000
    console.log "Middleware is runing on port * 7000"
    io = require 'socket.io-client'
    socket = io.connect 'http://localhost:8080', {reconnect: true, forceNew: true}
    socket.on 'connect', (socket) ->
       console.log "Connected! with server port * 8080"
    socket.on 'disconnect', ()->
       console.log 'Server is disconnected!'
       socket.close()

Validate DID for caller

    socket.on 'validateDID', (response) ->
      UniqueID = response.UniqueID
      arrUserInfo = globalObj.call.data[UniqueID]
      arrUserInfo.DidResponse = response
      arrDdidResp =arrUserInfo.DidResponse
      response ={}
      console.log arrDdidResp.msg
      if arrDdidResp.status is "OK"
         eventHandler "play_and_get_digits","1 11 3 5000 # #{ivrBase}/conference/8000/conf-pin.wav", UniqueID
      else
         eventHandler "playback", "{uniqueID=#{UniqueID},DidiValidate}#{ivrBase}/DIDdeactivate.wav"

Validate PIN from DTMF

    socket.on 'validatePIN', (response) ->
      UniqueID = response.UniqueID
      arrUserInfo = globalObj.call.data[UniqueID]
      #console.log "Arr Data === #{JSON.stringify(arrUserInfo, null, "    ")}"
      arrUserInfo.DTMFResponse = response
      arrResponse = arrUserInfo.DTMFResponse
      response ={}
      if arrResponse.status is "OK"
         eventHandler 'playback', "{uniqueID=#{UniqueID},PinAccepted}#{ivrBase}/PinAcceptJoinConfNow.wav"
      else
         eventHandler "playback","{uniqueID=#{UniqueID},InvalidPin}#{ivrBase}/ivr/8000/ivr-that_was_an_invalid_entry.wav"

Add Participiant user when confrence is established

    socket.on 'addParticipant', (response) ->
       UniqueID = response.UniqueID
       arrUserInfo = globalObj.call.data[UniqueID]
       if response.status is 'OK' && globalObj.data['Unique-ID'] is UniqueID
            arrUserInfo.JoinParticipantId = response.participant_id
            #console.log "Join Participant ID #{arrUserInfo.JoinParticipantId} and unique ID is #{globalObj.data['Unique-ID']}"
            arrDTMFResponse = arrUserInfo.DTMFResponse
            if arrDTMFResponse.ac_conf.recording is 1
                 now = new Date()
                 curentDate = dateFormat(now, "yyyy_mm_dd")
                 RecFileName= "#{arrDTMFResponse.ac_conf.id}_#{arrDTMFResponse.ac_conf.id_cc_card}_#{curentDate}_#{response.initial_user}"
                 path="/usr/local/freeswitch/recordings/#{RecFileName}.wav"
                 arrUserInfo.RecordingFilePath =path
                 if response.new_user is 1
                     recordingJsonObj = {
                       reqType: "recordingSaved",
                       UniqueID: arrDTMFResponse.UniqueID,
                       module: "AC",
                       arrData:{
                          location: path,
                          type: 0,
                          start_time: dateFormat(now, "yyyy-mm-dd HH:MM:ss"),
                          id_ac_conf: arrDTMFResponse.ac_conf.id,
                          server_ip: '192.168.1.230',
                          duration: 0
                          }
                        }
                 eventHandler "set","{uniqueID=#{UniqueID},ParticipantAdded}"
            else
                 eventHandler "set","{uniqueID=#{UniqueID},ParticipantAdded}"
