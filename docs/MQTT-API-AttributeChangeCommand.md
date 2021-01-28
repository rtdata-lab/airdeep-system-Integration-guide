# MQTT API - Airdeep.Attribute Change Command

## **설명**
  - 단말에서 Client Attribute 속성을 변경한다. (현재 firmwareVersion, lastBootingTime 만 존재하에 설명한다)
  - 서버에서 Shard Attribute 속성을 변경하면, 단말에게 변경된 속성 값이 전달되고 단말은 변경된 속성 값을 반영처리 한다. (현재 uploadFrequency 만 존재하에 설명한다)

## **선행 조건**
  - 장치 등록 
  - 디바이스 등록 과정에서 수신받은 MQTT 호스트 정보
  - Access Token (deviceCredentialsId)

</br>

## **Protocol**
- MQTT

</br>

## **Host Name**
- [디바이스 등록 과정](./HTTP-API-DeviceRegister.md)에서 수신받은 MQTT 호스트 정보

</br>

## **MQTT TOPIC**
NO | TOPIC    | Publish | Subscribe | Description
:------: | :------ | :---------: | :----------: | :----------
1  | v1/devices/me/attributes  | O | O | **Publish**</br> 'Client Attribute' 를 업데이트 한다.  </br> **Subscribe** </br> 'Shard Attribute' 업데이트 수신
2  | v1/devices/me/attributes/request/1  | O | X |  **Publish**</br> 서버에 저장된 Attribute(Client, Shard) 값을 요청한다. 
3  | v1/devices/me/attributes/response/1  | X | X | 2번 요청의 응답 값</br> on_message 콜백으로 응답 값이 수신된다. 

</br>

## **Payload**
NO | TOPIC    | Send(Publish) | Recv(on_message)
:------: | :------ | :--------- | :----------
1  | v1/devices/me/attributes  | { 'firmwareVersion': 5.0, 'lastBootingTime': timestamp(sec) } | { "uploadFrequency": 2 } 
2  | v1/devices/me/attributes/request/1  | { "clientKeys": "firmwareVersion, lastBootingTime", "sharedKeys": "uploadFrequency" } | X
3  | v1/devices/me/attributes/response/1  | X | { "firmwareVersion": 5.0,"lastBootingTime": timestamp(sec) },"shared":{"uploadFrequency": 1} }

</br>

## **Payload 설명**
Key        |  Description | Notes
:----------|:-----------------|:------------------
uploadFrequency| 변경된 Telemetry Upload 주기를 수신| 단말은 업로드 주기를 변경한다.
lastBootingTime| 단말의  부팅 시간 (Unix Timestamp, Sec)| 

</br></br>


## **시퀀스 다이어그램**

1. [Client-Side] firmwareVersion, lastBootingTime
    - 장비 업데이트 후 펌웨어 값 전달
   ```plantuml
   @startuml
     participant "AirDeep[AQS]" order 1
     participant "AirDeep[MQTT Server]" order 2
     hide footbox

     group 1. MQTT - Connection
      "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : MQTT Connection(MQTT_HOST, MQTT_PORT, ACCESS_TOKEN(deviceCredentialsId))
      "AirDeep[AQS]" <- "AirDeep[MQTT Server]"  : Ack(Connected)
     end

     == MQTT Session Established ==
     group 2. MQTT - 서버의 Attribute 값 요청 및 수신
     "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 1.1) Publish - ATTRIBUTES_REQUEST_TOPIC(서버에 저장된 Attribute(Client, Shard) 값 요청)
     "AirDeep[AQS]" <- "AirDeep[MQTT Server]" : 1.2) ATTRIBUTES_RESPONSE_TOPIC(Attribute (Client(firmwareVersion, lastBootingTime), Shard(uploadFrequency)) )
     end

     group 3. MQTT - Update Client Attribute (firmwareVersion, lastBootingTime)
     "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 2.1) Publish - ATTRIBUTES_TOPIC
     end

     group 4. MQTT - Upload Telemetry Data
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 3.1) Publish Data - (TELEMETRY_TOPIC) 
     end
     
     == MQTT Session Established ==
   @enduml
   ```
</br></br>

2. [Server-Side] uploadFrequency - Telemetry Upload Interval Time(Sec) 변경
    - 업로드 주기를 서버에서 변경

   ```plantuml
   @startuml
     participant "AirDeep[AQS]" order 1
     participant "AirDeep[MQTT Server]" order 2
     hide footbox

     group 1. MQTT - Connection
      "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : MQTT Connection(MQTT_HOST, MQTT_PORT, ACCESS_TOKEN(deviceCredentialsId))
      "AirDeep[AQS]" <- "AirDeep[MQTT Server]"  : Ack(Connected)
     end

     == MQTT Session Established ==
     group 2. MQTT - Shared Attribute 값 수신 후 적용
     "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 1.1) Publish - ATTRIBUTES_REQUEST_TOPIC(서버에 저장된 Attribute(Client, Shard) 값 요청)
     "AirDeep[AQS]" <- "AirDeep[MQTT Server]" : 1.2) ATTRIBUTES_RESPONSE_TOPIC(Attribute (Client(firmwareVersion, lastBootingTime), Shard(uploadFrequency)) )
     
     "AirDeep[AQS]" <- "AirDeep[AQS]" : 1.3) on_message(uploadFrequency (10 Sec))
     end

     group 3. MQTT - Subscribe
     "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 2.1) Subscribe - ATTRIBUTES_TOPIC
     
     end

    group 4. MQTT - Upload Telemetry Data
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 3.1) Publish Data - (TELEMETRY_TOPIC) 
     end
     
     group 5. MQTT - Update 'Shared Attribute'
       "AirDeep[MQTT Server]" <- "AirDeep[MQTT Server]" : 4.1) Change Shard Attr(uploadFrequency(30sec))
     end

     group 6. MQTT - on_message
       "AirDeep[AQS]" <- "AirDeep[MQTT Server]" : 4.2) Changed Data(ATTRIBUTES_TOPIC)
       "AirDeep[AQS]" <- "AirDeep[AQS]" : 4.3) Update local uploadFrequency (30 Sec)
       note left: 변경된 Shared Attribute 수신
     end

     group 7. MQTT - Upload Telemetry Data (uploadFrequency: 30 Sec)
       "AirDeep[AQS]" -> "AirDeep[MQTT Server]" : 5.1) Publish Data - (TELEMETRY_TOPIC,{sensor_data})
       note left: 변경된 Shared Attribute 적용
     end
     == MQTT Session Established ==
   @enduml
   ```

</br>


 
## **Python Example**
```python
import requests
import json
from datetime import datetime

uploadFrequency = 1

ATTRIBUTES_TOPIC = "v1/devices/me/attributes"
ATTRIBUTES_REQUEST_TOPIC = "v1/devices/me/attributes/request/1"
ATTRIBUTES_RESPONSE_TOPIC = "v1/devices/me/attributes/response/1"

# When device is connected to AirDeep server
def on_connect(client, userdata, rc, *extra_params):

    # 펍웨어 버전이 업데이트 되었다면, 장비의 펌웨어 버전을 전달한다.
    device_client_attributes = {
        'firmwareVersion': 5.0, 'lastBootingTime': timestamp
    }
    client.publish(ATTRIBUTES_TOPIC, json.dumps(device_client_attributes))


    # 1.1) Attribute(Client, Shared) 값 요청한다.
    client.publish(ATTRIBUTES_REQUEST_TOPIC , json.dumps({"clientKeys": "firmwareVersion,lastBootingTime","sharedKeys": "uploadFrequency"}))

    # 2.1) 서버에서 Device의 Shared Attribute 변경 시, 변경된 값을 받기 위하여 Subscribe 요청.
    client.subscribe(ATTRIBUTES_TOPIC)


def on_message(client, userdata, msg):
    
    global uploadFrequency

    payload = json.loads(msg.payload)

    # 1.2) 서버로 부터 요청한 Attribute(Client, Shard) 값을 수신한다.
    if msg.topic == ATTRIBUTES_RESPONSE_TOPIC:
        if payload["shared"]["uploadFrequency"]  > 0:
            uploadFrequency = payload["shared"]["uploadFrequency"]
        return

    # 2.2) 서버에서 Shard Attribute 변경 시 수신된다.
    if msg.topic == ATTRIBUTES_TOPIC:
        if payload["uploadFrequency"] > 0:
            uploadFrequency = payload["uploadFrequency"]
        return


    
# Sensor data
sensor_data = {
    'temperature': 0, 'temperature_max': 0,'temperature_unit':'C', 
    'humidity': 0, 'humidity_max': 0, 'humidity_unit': "%",
    'co2': 0, 'co2_max': 0, 'co2_unit': "ppm",
    'pm10': 0, 'pm10_max': 0, 'pm10_unit': "µg/m3", 
    'pm2.5': 0, 'pm2.5_max': 0, 'pm2.5_unit': "µg/m3", 
    'tvoc': 0, 'tvoc_max': 0, 'tvoc_unit': "mg/m3",
    'report_reason': 'Report Reason'
}

# publish sensor data
try:
    while True:
        # Set sensor data
        sensor_data['temperature'] = random.randint(0, 100)
        sensor_data['humidity'] = random.randint(0, 100)
        sensor_data['humidity_max'] = random.randint(0, 100)
        sensor_data['co2'] = random.randint(0, 100)
        sensor_data['co2_max'] = random.randint(0, 100)
        sensor_data['pm10'] = random.randint(0, 100)
        sensor_data['pm10_max'] = random.randint(0, 100)
        sensor_data['pm2.5'] = random.randint(0, 100)
        sensor_data['pm2.5_max'] = random.randint(0, 100)
        sensor_data['tvoc'] = random.randint(0, 100)
        sensor_data['tvoc_max'] = random.randint(0, 100)

        client.publish(TELEMETRY_TOPIC, json.dumps(sensor_data), 1)
        time.sleep(uploadFrequency) 


  ```


