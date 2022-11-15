import paho.mqtt.client as mqtt
from InboundMessage_pb2 import *
from OutboundMessage_pb2 import *
import sys
import json
from datetime import datetime
from time import sleep
import re
import os
import tkinter as tk

# Default Values
#in_client_name = "Client-Vehicle"
#out_client_name = "Client-FMS"
#host_name = "127.0.0.1"
#subOutTopic = "OCTA/FMS/1/OTA2"
#publishInTopic = "OCTA/FMS/1/OTA2"
#subInTopic = "OCTA/VEHICLE/1234/OTA2"
#publishOutTopic = "OCTA/VEHICLE/1234/OTA2"
#qos = 1
#keepAliveTime = 600

# Loading Configuration Values
f = open(sys.argv[1], 'r')
data = json.load(f)

in_client_name = data["clientInfo"]["InClientName"]
out_client_name = data["clientInfo"]["OutClientName"]
host_name = data["clientInfo"]["hostName"]
subOutTopic = data["clientInfo"]["subOutTopic"]
publishOutTopic = data["clientInfo"]["pubOutTopic"]
publishInTopic = data["clientInfo"]["pubInTopic"]
subInTopic = data["clientInfo"]["subInTopic"]
portConnection = data["clientInfo"]["port"]
maxPacketSize = data["clientInfo"]["max_payload_len"]
subqos = data["clientInfo"]["sub_qos"]
pubqos = data["clientInfo"]["pub_qos"]
retain_msg = data["clientInfo"]["retain_msg"]
keepAliveTime = data["clientInfo"]["keep_alive"]
cleanSession = data["clientInfo"]["clean_session"]
timeout = data["clientInfo"]["timeout"]

f.close()

messageInTypes = {}
messageOutTypes = {}
vehicleTopics = []
vehicleID = 1000
transmissionID = 0
filenameIn = ""
foldernameIn = ""
filenameOut = ""
foldernameOut = ""
graphical = False
clientIn = clientOut = ""

# @param: boolean whether inbound or not, message number
# @return: index of message based on .json
def getMessageIndex(inbound, msgNum):
    f = open(sys.argv[1], 'r')
    data = json.load(f)
    indexName = "InboundOTAMessage"
    index = -1

    if not inbound:
        indexName = "OutboundOTAMessage"

    for i in range(len(data[indexName])):
        if data[indexName][i]["CadId"] == str(msgNum):
            index = i

    f.close()
    return index


# Creates new outbound packet obj using header from json
# @param: None
# @return: new packet obj
def newOutboundPacket():
    packet = OutboundPacket()
    packet.PktHeader.MessageCount = 1
    packet.PktHeader.Outage = False
    return packet


# Adds message onto outbound packet obj
# @param: packet object to add onto, message to add, nondefault parameter values dictionary
# @return: modified packet object
def addMsgToOutboundPacket(packet, msgNum, params):
    f = open(sys.argv[1], 'r')
    data = json.load(f)
    index = getMessageIndex(False, msgNum)

    if index == -1:
        print("Outbound message id", msgNum, "does not exist.")
        exit()

    messageTemp = packet.PktMessage.add()
    messageTemp.MsgHeader.TypeID = int(msgNum)
    #messageTemp.MsgHeader.ExtendedIDFlag = False
    #messageTemp.MsgHeader.Std_MessageID = 0
    #messageTemp.MsgHeader.Extended_MessageID = bytes("000")
    message = data["OutboundOTAMessage"][index]["Parameter"]

    for param, value in params.items():
        found = False

        for p in message:
            if param in p.values():
                found = True
                p["ParamValue"] = value
                break

        if not found:
            print("Parameter", param, "not found in message type", msgNum)

    messageTemp.TheMessage = bytes(str(message), 'utf-8')

    f.close()
    return packet


# Creates new inbound packet obj using header from json
# @param: None
# @return: new packet obj
def newInboundPacket(vehicleID):
    global transmissionID
    f = open(sys.argv[1], 'r')
    data = json.load(f)

    packet = InboundPacket()
    for item in data["Header"]:
        packet.PktHeader.VehicleID = vehicleID
        packet.PktHeader.TransmissionID = transmissionID + 1
        packet.PktHeader.MessageCount = 1
        #packet.PktHeader.GPS_NULL_Indicator = False
        packet.PktHeader.CompressGPSHeading = item["cGPSHeading"]
        packet.PktHeader.CompressGPSLat = item["cGPSLat"]
        packet.PktHeader.CompressGPSLon = item["cGPSLon"]
        packet.PktHeader.Speed = bytes(item["Speed"], 'utf-8')
        packet.PktHeader.OffRoute = item["OffRoute"]
        packet.PktHeader.LateIndicator = item["LateIndicator"]
        packet.PktHeader.SchedDeviation = bytes(item["SchedDeviation"],
                                                'utf-8')
        packet.PktHeader.GPSHeading = item["GPSHeading"]
        packet.PktHeader.GPSLat = item["GPSLat"]
        packet.PktHeader.GPSLon = item["GPSLon"]
        packet.PktHeader.Altitude = item["Altitude"]

    f.close()
    return packet


# Adds message onto inbound packet obj
# @param: packet object to add onto, message to add, nondefault parameter values dictionary
# @return: modified packet object
def addMsgToInboundPacket(packet, msgNum, params):
    f = open(sys.argv[1], 'r')
    data = json.load(f)
    index = getMessageIndex(True, msgNum)

    if index == -1:
        print("Inbound message id", msgNum, "does not exist.")
        exit()

    messageTemp = packet.PktMessage.add()
    messageTemp.MsgHeader.TypeID = int(msgNum)
    #messageTemp.MsgHeader.ExtendedIDFlag = False
    #messageTemp.MsgHeader.Std_MessageID = 0
    #messageTemp.MsgHeader.Extended_MessageID = bytes("000")
    message = data["InboundOTAMessage"][index]["Parameter"]

    for param, value in params.items():
        found = False

        for p in message:
            if param in p.values():
                found = True
                p["ParamValue"] = value
                break

        if not found:
            print("Parameter", param, "not found in message type", msgNum)

    messageTemp.TheMessage = bytes(str(message), 'utf-8')

    f.close()
    return packet


def processInstruction(user_str, readFile, labels, client, vehicleID):
    global transmissionID

    sendLex = re.match(r"^Send (\d+|[a-zA-Z]+)(\s*[a-zA-Z]+=\d+\s*)*$", user_str)
    delayLex = re.match(r"^Delay (\d+)$", user_str)
    labelLex = re.match(r"^([a-zA-Z]+):$", user_str)
    loopLex = re.match(r"^Loop ([a-zA-Z]+) (\d+)$", user_str)

    if sendLex != None:
        sendID = sendLex[1].isdigit() and sendLex[1] in messageInTypes.keys()
        sendStr = sendLex[1] in messageInTypes.values()
        if sendID or sendStr:
            params = re.findall(r"[a-zA-Z]+=\d+", user_str)
            paramVals = {}

            for param in params:
                p = re.match(r"([a-zA-Z]+)=(\d+)", param)
                paramVals[p.group(1)] = p.group(2)
            
            messageNum = sendLex[1]
            if sendStr:
                for key,value in messageInTypes.items():
                    if sendLex[1] in value:
                        messageNum = key
                
            packet = addMsgToInboundPacket(newInboundPacket(vehicleID), messageNum, paramVals)

            transmissionID += 1
            transmissionID %= 63

            theMessage = packet.SerializeToString()
            infot = client.publish(publishInTopic, payload=theMessage, qos=pubqos)
            if infot.is_published:
                print("Published successfully.")
        else:
            print("The only messages that can be sent are:", messageInTypes.items())
    elif delayLex != None:
        print("Waiting", delayLex[1], "seconds")
        sleep(int(delayLex[1]))
    elif labelLex != None:
        if labelLex[1] in labels.keys():
            print("Label already defined!")
        else:
            labels[labelLex[1]] = list()
            print(labels)
    elif loopLex != None:
        if loopLex[1] not in labels.keys():
            print("Label", loopLex[1], "not defined.")
        elif "End" not in labels.get(loopLex[1]):
            print("Label", loopLex[1], "not ended.")
        else:
            instructionArr = labels.get(loopLex[1])
            for i in range(int(loopLex[2])):
                instructionNum = 0
                while instructionArr[instructionNum] != "End":
                    processInstruction(instructionArr[instructionNum], readFile, labels, client, vehicleID)
                    instructionNum += 1
    else:
        print("Lexing error!")


currInbound = newInboundPacket(vehicleID)
currOutbound = newOutboundPacket()

def main():
    global clientIn, clientOut, filenameIn, filenameOut, in_client_name, subInTopic, publishOutTopic, foldernameIn, foldernameOut, vehicleID, currInbound, currOutbound

    def on_message_out(client, userdata, message):
        global filenameOut, vehicleTopics
        write = open(filenameOut, 'a')

        msg_received = message.payload
        #print(type(msg_received))
        #print("message received (serialized):", msg_received, "\n")
        packet = InboundPacket()
        packet.ParseFromString(msg_received)
        current_time = datetime.now().strftime("%H:%M:%S")
        write.write("Message received at " + current_time + ":\n" + str(packet))
        
        if graphical:
            outputText['state'] = tk.NORMAL
            outputText.insert(tk.END, current_time + ": Message received from vehicle " + str(packet.PktHeader.VehicleID) + "\n" + str(packet))
            outputText['state'] = tk.DISABLED
        else:
            print("Message received at", current_time + ":\n", packet, end="")

        f = open(sys.argv[1], 'r')
        data = json.load(f)

        topic = "OCTA/VEHICLE/" + str(packet.PktHeader.VehicleID) + "/OTA2"

        vehicleTopics.append(topic)

        for msg in packet.PktMessage:
            index = getMessageIndex(True, msg.MsgHeader.TypeID)

            for response in data["InboundOTAMessage"][index]["Response"]:
                p = addMsgToOutboundPacket(newOutboundPacket(), str(response), {})
                theMessage = p.SerializeToString()
                client.publish(topic, payload=theMessage, qos=pubqos)

                write.write("Sending response message " + str(response) + "\n")

                if graphical:
                    current_time = datetime.now().strftime("%H:%M:%S")

                    outputText['state'] = tk.NORMAL
                    outputText.insert(tk.END, current_time + ": Sending response message " + str(response) + "\n")
                    outputText['state'] = tk.DISABLED
                else:
                    print("Sending response message", str(response))
                

        f.close()


    def on_message_in(client, userdata, message):
        global filenameIn
        write = open(filenameIn, 'a')
        msg_received = message.payload
        #print(type(msg_received))
        #print("message received (serialized):", msg_received, "\n")
        packet = OutboundPacket()
        packet.ParseFromString(msg_received)
        current_time = datetime.now().strftime("%H:%M:%S")
        write.write("Message received at " + current_time + ":\n" + str(packet))
        if graphical:
            outputText['state'] = tk.NORMAL
            outputText.insert(tk.END, current_time + ": Message received\n" + str(packet))
            outputText['state'] = tk.DISABLED
        else:
            print("Message received at", current_time + ":\n", packet, end="")


    labels = {}
    readFile = False
    minArgs = 3

    if len(sys.argv) == minArgs:
        readFile = False
    elif len(sys.argv) == minArgs + 1:
        readFile = True
        f2 = open(sys.argv[3], 'r')
    else:
        print("Command line argument error. Usage: python", sys.argv[0],
              ".json in####/out/graphical .txt")
        exit()

    f = open(sys.argv[1], 'r')
    data = json.load(f)

    for i in data["InboundOTAMessage"]:
        messageInTypes[i["CadId"]] = i["Name"]

    for i in data["OutboundOTAMessage"]:
        messageOutTypes[i["CadId"]] = i["Name"]

    f.close()

    reVehicle = re.match(r"^in(\d*)$", sys.argv[2])

    if "graphical" in sys.argv:
        global graphical, inboundBool

        def disableRoot():
            inboundCheck['state'] = tk.DISABLED
            vehicleIDField['state'] = tk.DISABLED
            messageTypeMenu['state'] = tk.DISABLED
            paramsButton['state'] = tk.DISABLED
            sendButton['state'] = tk.DISABLED


        def enableRoot():
            inboundCheck['state'] = tk.NORMAL
            vehicleIDField['state'] = tk.NORMAL
            messageTypeMenu['state'] = tk.NORMAL
            paramsButton['state'] = tk.NORMAL
            sendButton['state'] = tk.NORMAL


        def paramsWindow():
            global inboundBool

            def closeParams():
                enableRoot()
                window.destroy()


            def confirmParams():
                global currInbound, inboundBool, currOutbound
                # Set params here...
                params = {}
                for i in range(len(grid)):
                    params[grid[i][0]['text']] = grid[i][1].get()

                if inboundBool:
                    currInbound = addMsgToInboundPacket(newInboundPacket(vehicleID), currInbound.PktMessage[0].MsgHeader.TypeID, params)

                    current_time = datetime.now().strftime("%H:%M:%S")

                    outputText['state'] = tk.NORMAL
                    outputText.insert(tk.END, current_time + ": Message " + InMessageHeader.InboundMessageTypeNos.Name(currInbound.PktMessage[0].MsgHeader.TypeID) + " parameters set\n")
                    outputText['state'] = tk.DISABLED
                else:
                    currOutbound = addMsgToOutboundPacket(newOutboundPacket(), currOutbound.PktMessage[0].MsgHeader.TypeID, params)

                    current_time = datetime.now().strftime("%H:%M:%S")

                    outputText['state'] = tk.NORMAL
                    outputText.insert(tk.END, current_time + ": Message " + OutMessageHeader.OutboundMessageTypeNos.Name(currOutbound.PktMessage[0].MsgHeader.TypeID) + " parameters set\n")
                    outputText['state'] = tk.DISABLED

                enableRoot()
                window.destroy()


            disableRoot()
            indexName = "InboundOTAMessage"
            if not inboundBool: # inbound checkbox not checked
                indexName = "OutboundOTAMessage"
                for key, value in messageOutTypes.items():
                    if messageTypeVar.get() in value:
                        messageNum = key
            else:
                for key, value in messageInTypes.items():
                    if messageTypeVar.get() in value:
                        messageNum = key

            index = getMessageIndex(inboundBool, messageNum)

            window = tk.Toplevel()
            window.title("Message " + messageTypeVar.get() + " Parameters")
            window.protocol("WM_DELETE_WINDOW", closeParams)

            paramsFrame = tk.Frame(window)

            parameters = data[indexName][index]["Parameter"]

            grid = [[-1 for _ in range(2)] for _ in range(len(parameters))]

            for i in range(len(parameters)):
                grid[i][0] = tk.Label(paramsFrame, text=parameters[i]["ParamName"])
                grid[i][1] = tk.Entry(paramsFrame)
                grid[i][1].insert(0, parameters[i]["ParamValue"])

            confirmButton = tk.Button(paramsFrame, text="Confirm Parameters", command=confirmParams)

            paramsFrame.pack()

            for i in range(len(grid)):
                for j in range(len(grid[i])):
                    grid[i][j].grid(row=i, column=j)

            confirmButton.grid(row=len(grid), column=0, columnspan=2)


        def inboundSelect():
            global inboundBool, clientOut, clientIn, vehicleTopics
            if inbound.get() == 1: # inbound checkbox is checked
                inboundBool = True
                vehicleIDField['state'] = tk.NORMAL
                root.title("OTAV2 Simulator Vehicle " + str(vehID.get()))
                messageTypeMenu['menu'].delete(0, 'end')

                for val in messageInTypes.values():
                    messageTypeMenu['menu'].add_command(label=val, command=tk._setit(messageTypeVar, val))

                messageTypeVar.set(next(iter(messageInTypes.values())))

                current_time = datetime.now().strftime("%H:%M:%S")

                outputText['state'] = tk.NORMAL
                outputText.delete(1.0, tk.END)
                outputText.insert(tk.END, current_time + ": Vehicle connected to MQTT broker(" + host_name + ")\n")
                outputText['state'] = tk.DISABLED

                clientOut.disconnect()

                clientIn = mqtt.Client(client_id=in_client_name, clean_session=cleanSession)
                clientIn.on_message = on_message_in
                clientIn.connect(host_name, keepalive=keepAliveTime, port=portConnection)
                clientIn.subscribe(subInTopic, qos=subqos)
                clientIn.loop_start()

            else: # inbound checkbox not checked
                vehicleTopics = []
                inboundBool = False
                vehicleIDField['state'] = tk.DISABLED
                root.title("OTAV2 Simulator Fixed End")
                messageTypeMenu['menu'].delete(0, 'end')

                for val in messageOutTypes.values():
                    messageTypeMenu['menu'].add_command(label=val, command=tk._setit(messageTypeVar, val))

                messageTypeVar.set(next(iter(messageOutTypes.values())))

                current_time = datetime.now().strftime("%H:%M:%S")

                outputText['state'] = tk.NORMAL
                outputText.delete(1.0, tk.END)
                outputText.insert(tk.END, current_time + ": Fixed end connected to MQTT broker(" + host_name + ")\n")
                outputText['state'] = tk.DISABLED

                clientIn.disconnect()

                clientOut = mqtt.Client(client_id=out_client_name, clean_session=cleanSession)
                clientOut.on_message = on_message_out
                clientOut.connect(host_name, keepalive=keepAliveTime, port=portConnection)
                clientOut.subscribe(subOutTopic, qos=subqos)
                clientOut.loop_start()


        def changeVehicleID(*args):
            global vehicleID, in_client_name, subInTopic, publishOutTopic, foldernameIn, filenameIn, clientIn, currInbound

            try:
                vehicleID = int(vehicleIDField.get())
            except ValueError:
                vehicleID = 1000

            if vehicleID >= 1000:
                root.title("OTAV2 Simulator Vehicle " + vehicleIDField.get())

                in_client_name = "Client-Vehicle" + str(vehicleID)
                subInTopic = "OCTA/VEHICLE/" + str(vehicleID) + "/OTA2"
                publishOutTopic = "OCTA/VEHICLE/" + str(vehicleID) + "/OTA2"
                foldernameIn = r"logs\vehicle\\" + datetime.now().strftime('%d-%m-%Y-%H-%M')

                if not os.path.exists(foldernameIn):
                        os.mkdir(foldernameIn)
                filenameIn = foldernameIn + "\\" + str(vehicleID) + ".txt"
                open(filenameIn, 'w')

                clientIn.disconnect()

                for key, value in messageInTypes.items():
                    if messageTypeVar.get() in value:
                        currInbound = addMsgToInboundPacket(newInboundPacket(vehicleID), key, {})

                clientIn = mqtt.Client(client_id=in_client_name, clean_session=cleanSession)
                clientIn.on_message = on_message_in
                clientIn.connect(host_name, keepalive=keepAliveTime, port=portConnection)

                clientIn.loop_start()

                clientIn.subscribe(subInTopic, qos=subqos)


        def sendMessage():
            global transmissionID, currInbound, currOutbound, inboundBool

            if inboundBool:
                transmissionID += 1
                transmissionID %= 63

                theMessage = currInbound.SerializeToString()
                clientIn.publish(publishInTopic, payload=theMessage, qos=pubqos)

                current_time = datetime.now().strftime("%H:%M:%S")

                outputText['state'] = tk.NORMAL
                outputText.insert(tk.END, current_time + ": Vehicle sent message " + InMessageHeader.InboundMessageTypeNos.Name(currInbound.PktMessage[0].MsgHeader.TypeID) + "\n")
                outputText['state'] = tk.DISABLED

                for key, value in messageInTypes.items():
                    if messageTypeVar.get() in value:
                        currInbound = addMsgToInboundPacket(newInboundPacket(vehicleID), key, {})
            else:
                theMessage = currOutbound.SerializeToString()

                for topic in vehicleTopics:
                    clientOut.publish(topic, payload=theMessage, qos=pubqos)

                current_time = datetime.now().strftime("%H:%M:%S")

                outputText['state'] = tk.NORMAL
                outputText.insert(tk.END, current_time + ": Fixed end sent message " + OutMessageHeader.OutboundMessageTypeNos.Name(currOutbound.PktMessage[0].MsgHeader.TypeID) 
                                  + " to " + str(len(vehicleTopics)) + " vehicles\n")
                outputText['state'] = tk.DISABLED

                for key, value in messageOutTypes.items():
                    if messageTypeVar.get() in value:
                        currOutbound = addMsgToOutboundPacket(newOutboundPacket(), key, {})

        
        def changeMessage(*args):
            global currInbound, currOutbound, inboundBool

            if inboundBool:
                for key, value in messageInTypes.items():
                    if messageTypeVar.get() in value:
                        currInbound = addMsgToInboundPacket(newInboundPacket(vehicleID), key, {})
            else:
                for key, value in messageOutTypes.items():
                    if messageTypeVar.get() in value:
                        currOutbound = addMsgToOutboundPacket(newOutboundPacket(), key, {})


        graphical = True
        inboundBool = True
        root = tk.Tk()

        mainFrame = tk.Frame(root)

        inbound = tk.IntVar()
        inboundCheck = tk.Checkbutton(mainFrame, text="Inbound", variable=inbound, command=inboundSelect)
        inboundCheck.select()

        vehicleIDLabel = tk.Label(mainFrame, text="Vehicle ID")

        vehID = tk.IntVar()
        vehID.set(vehicleID) # Default Vehicle ID
        vehID.trace("w", callback=changeVehicleID)
        vehicleIDField = tk.Entry(mainFrame, textvariable=vehID)

        messageTypeLabel = tk.Label(mainFrame, text="Message Type")

        messages = messageInTypes.values()
        messageTypeVar = tk.StringVar()
        messageTypeVar.set(next(iter(messages)))
        messageTypeVar.trace('w', changeMessage)
        messageTypeMenu = tk.OptionMenu(mainFrame, messageTypeVar, *messages, command=changeMessage)

        paramsButton = tk.Button(mainFrame, text="Set Parameters", command=paramsWindow)

        sendButton = tk.Button(mainFrame, text="Send Message", command=sendMessage)

        consoleLabel = tk.Label(mainFrame, text="Console")

        textFrame = tk.Frame(mainFrame)

        outputText = tk.Text(textFrame)

        scrollBar = tk.Scrollbar(textFrame, command=outputText.yview)
        outputText['yscrollcommand'] = scrollBar.set

        mainFrame.pack()

        root.title("OTAV2 Simulator Vehicle " + str(vehID.get()))

        inboundCheck.grid(row=0, column=0)
        vehicleIDLabel.grid(row=0, column=1)
        vehicleIDField.grid(row=0, column=2)
        messageTypeLabel.grid(row=1, column=0)
        messageTypeMenu.grid(row=1, column=1)
        paramsButton.grid(row=1, column=2)
        sendButton.grid(row=2, column=1)
        consoleLabel.grid(row=3, column=0, sticky=tk.W)
        outputText.pack(side=tk.LEFT, fill=tk.BOTH)
        scrollBar.pack(side=tk.RIGHT, fill=tk.Y)
        textFrame.grid(row=4, column=0, columnspan=3)

        messages = messageInTypes.keys()
        currInbound = addMsgToInboundPacket(currInbound, next(iter(messages)), {})

        messages = messageOutTypes.keys()
        currOutbound = addMsgToOutboundPacket(currOutbound, next(iter(messages)), {})


        in_client_name += str(vehicleID)
        subInTopic = "OCTA/VEHICLE/" + str(vehicleID) + "/OTA2"
        publishOutTopic = "OCTA/VEHICLE/" + str(vehicleID) + "/OTA2"
        foldernameIn = r"logs\vehicle\\" + datetime.now().strftime('%d-%m-%Y-%H-%M')

        if not os.path.exists(foldernameIn):
             os.mkdir(foldernameIn)
        filenameIn = foldernameIn + "\\" + str(vehicleID) + ".txt"
        open(filenameIn, 'w')

        current_time = datetime.now().strftime("%H:%M:%S")

        outputText['state'] = tk.NORMAL
        outputText.insert(tk.END, current_time + ": Vehicle connected to MQTT broker(" + host_name + ")\n")
        outputText['state'] = tk.DISABLED

        current_time = datetime.now().strftime("%H:%M:%S")

        #outputText2['state'] = tk.NORMAL
        #outputText2.insert(tk.END, current_time + ": Fixed end connected to MQTT broker(" + host_name + ")\n")
        #outputText2['state'] = tk.DISABLED

        # mqtt
        clientIn = mqtt.Client(client_id=in_client_name, clean_session=cleanSession)
        clientIn.on_message = on_message_in
        clientIn.connect(host_name, keepalive=keepAliveTime, port=portConnection)

        filenameOut = r"logs\fms\\" + datetime.now().strftime('%d-%m-%Y-%H-%M.txt')
        open(filenameOut, 'w')
        clientOut = mqtt.Client(client_id=out_client_name, clean_session=cleanSession)
        clientOut.on_message = on_message_out
        
        #clientOut.connect(host_name, keepalive=keepAliveTime)
        #clientOut.subscribe(subOutTopic, qos=qos)

        #clientOut.loop_start()

        clientIn.loop_start()

        clientIn.subscribe(subInTopic, qos=subqos)

        root.mainloop()
    else:
        if reVehicle != None:

            if reVehicle[1] != "":
                vehicleID = int(reVehicle[1])

            in_client_name += str(vehicleID)
            subInTopic = "OCTA/VEHICLE/" + str(vehicleID) + "/OTA2"
            publishOutTopic = "OCTA/VEHICLE/" + str(vehicleID) + "/OTA2"
            foldernameIn = r"logs\vehicle\\" + datetime.now().strftime('%d-%m-%Y-%H-%M')
            if not os.path.exists(foldernameIn):
                os.mkdir(foldernameIn)

            filenameIn = foldernameIn + "\\" + str(vehicleID) + ".txt"
            open(filenameIn, 'w')
            write = open(filenameIn, 'a')

            # mqtt
            client = mqtt.Client(client_id=in_client_name, clean_session=cleanSession)
            client.on_message = on_message_in
            client.connect(host_name, keepalive=keepAliveTime, port=portConnection)

            client.loop_start()

            client.subscribe(subInTopic, qos=subqos)

            user_str = ""

            print("OTAV2 Simulator")

            while True:
                now = datetime.now()
                current_time = now.strftime("%H:%M:%S")

                #write.write(current_time + ">>>")
                print(current_time + ">>>", end='')
                #write.write(user_str)

                if readFile:
                    user_str = f2.readline()
                    print(user_str, end='')
                else:
                    user_str = str(input())


                if user_str == "quit":
                    client.disconnect()
                    break

                labelBool = False
                for key in labels:
                    if labels[key] != None and "End" not in labels[key]:
                        labelBool = True

                if labelBool:
                    endLabelLex = re.match(r"^End ([a-zA-Z]+)$", user_str)
                    if endLabelLex != None:
                        if endLabelLex[1] not in labels.keys():
                            print("Label does not exist.")
                        else:
                            labels.get(endLabelLex[1]).append("End")
                    else:
                        for key in labels:
                            if "End" not in labels[key]:
                                labels.get(key).append(user_str)
                else:
                    processInstruction(user_str, readFile, labels, client, vehicleID)

            client.loop_stop()
        elif sys.argv[2] == "out":
            filenameOut = r"logs\fms\\" + datetime.now().strftime('%d-%m-%Y-%H-%M.txt')
            open(filenameOut, 'w')
            client = mqtt.Client(client_id=out_client_name, clean_session=cleanSession)
            client.on_message = on_message_out
        
            client.connect(host_name, keepalive=keepAliveTime, port=portConnection)
            client.subscribe(subOutTopic, qos=subqos)

            client.loop_forever()
        else:
            print("Please specify inbound(in####) or outbound(out) messages.")


if __name__ == "__main__":
    main()
