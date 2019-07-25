# ESPNOW flooding mesh library

See example project: https://github.com/arttupii/EspNowFloodingMesh

ESPNOW flooding mesh library.

Features:
- Maximum number of slave nodes: unlimited
- Number of master nodes: 1
- Master node sends time sync message every 10s to all nodes. (clocks are syncronised)
- Every message has a time stamp. If the time stamp is too old (or from the future), the message will be rejected.
- All messages are crypted (AES128)
- Flooding mesh support
- TTL support (time to life)
- ESP32, ESP2866, ESP01
- Battery node support
- Request&Reply support


## Flooding mesh network
In this network example ttl must be >= 4
```
               SlaveNode
                   |
                   |         Message from master to BatteryNode
                   |   ---------------------------+
                   |                     ttl=4    |
SlaveNode-------MasterNode-------------SlaveNode  |
                   |                     |        |
                   |                     |        |
                   |                     |        |
                   |                     |        |
               SlaveNode                 |        |
                   |                     |        |
                   |                     |        |
                   |                     |        +------------------------------------------------>
                   |                     | ttl=3         ttl=2              ttl=1
SlaveNode-------SlaveNode-------------SlaveNode-------SlaveNode-------------SlaveNode---------BatteryNode
   |               |                     |
   |               |                     |
   |               |                     |
   |               |                     |
   +-----------SlaveNode-----------------+
```  
## Create master node:
```c++
#include <EspNowFloodingMesh.h>

#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00,0x11,0x22,0x33,0x44,0x55,0x66,0x77,0x88,0x99,0xAA,0xBB,0xCC,0xDD,0xEE, 0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len){
  if(len>0) {
    Serial.println((const char*)data);
  }
}

void setup() {
  Serial.begin(115200);

  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_secredkey(secredKey);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);
  espNowFloodingMesh_setToMasterRole(true,3); //Set ttl to 3. TIME_SYNC message use this ttl
  espNowFloodingMesh_ErrorDebugCB([](int level, const char *str) {
    Serial.println(str);
  });
}

void loop() {
  static unsigned long m = millis();
  if(m+5000<millis()) {
    char message[] = "MASTER HELLO MESSAGE";
    espNowFloodingMesh_send((uint8_t*)message, sizeof(message), 3); //set ttl to 3
    m = millis();
  }
  espNowFloodingMesh_loop();
  delay(10);
}
```
## Create slave node:
```c++
#include <EspNowFloodingMesh.h>

#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00,0x11,0x22,0x33,0x44,0x55,0x66,0x77,0x88,0x99,0xAA,0xBB,0xCC,0xDD,0xEE,0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len){
  if(len>0) {
    Serial.println((const char*)data);
  }
}

void setup() {
  Serial.begin(115200);

  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);
  espNowFloodingMesh_secredkey(secredKey);

  //Ask instant sync from master.
  espNowFloodingMesh_requestInstantTimeSyncFromMaster();
  espNowFloodingMesh_ErrorDebugCB([](int level, const char *str) {
    Serial.println(str);
  });
  while(espNowFloodingMesh_isSyncedWithMaster()==false);
}

void loop() {
  static unsigned long m = millis();
  if(m+5000<millis()) {
    char message[] = "SLAVE HELLO MESSAGE";
    espNowFloodingMesh_send((uint8_t*)message, sizeof(message), 3); //set ttl to 3
    m = millis();
  }
  espNowFloodingMesh_loop();
  delay(10);
}
```

## Create slave node (Battery):
```c++
#include <EspNowFloodingMesh.h>
#include <time.h>
#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00, 0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77, 0x88, 0x99, 0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len) {
  if (len > 0) {
    Serial.println((const char*)data);
  }
}

void setup() {
  Serial.begin(115200);
  //Set device in AP mode to begin with
  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);
  espNowFloodingMesh_secredkey(secredKey);
  espNowFloodingMesh_setToBatteryNode();
}

void loop() {
  static unsigned long m = millis();

  //Ask instant sync from master.
  espNowFloodingMesh_requestInstantTimeSyncFromMaster();
  while (espNowFloodingMesh_isSyncedWithMaster() == false);
  char message[] = "SLAVE(12) HELLO MESSAGE";
  espNowFloodingMesh_send((uint8_t*)message, sizeof(message), 0); //set ttl to 3
  espNowFloodingMesh_loop();
  ESP.deepSleep(60000, WAKE_RF_DEFAULT); //Wakeup every minute
}
```
## Send message and get reply:
Send "MARCO" to other nodes
```c++

#include <EspNowFloodingMesh.h>

#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00,0x11,0x22,0x33,0x44,0x55,0x66,0x77,0x88,0x99,0xAA,0xBB,0xCC,0xDD,0xEE, 0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len, uint32_t replyPrt){
}

void setup() {
  Serial.begin(115200);
  //Set device in AP mode to begin with
  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_secredkey(secredKey);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);
  espNowFloodingMesh_setToMasterRole(true,3); //Set ttl to 3.
  espNowFloodingMesh_ErrorDebugCB([](int level, const char *str) {
    Serial.println(str);
  });
}

void loop() {
  static unsigned long m = millis();
  if(m+5000<millis()) {
    char message2[] = "MARCO";
    espNowFloodingMesh_sendAndHandleReply((uint8_t*)message2, sizeof(message2),3,[](const uint8_t *data, int len){
        if(len>0) { //Handle reply from other node
          Serial.print("Reply: "); //Prinst POLO.
          Serial.println((const char*)data);
        }
    });
    m = millis();
  }
  espNowFloodingMesh_loop();
  delay(10);
}
```
Answer to "MARCO" and send "POLO"
```c++
#include <EspNowFloodingMesh.h>

#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00,0x11,0x22,0x33,0x44,0x55,0x66,0x77,0x88,0x99,0xAA,0xBB,0xCC,0xDD,0xEE, 0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len, uint32_t replyPrt){
  if(len>0) {
    if(replyPrt) { //Reply asked. Send reply
        char m[]="POLO";
        Serial.println((char*)data); //Prints MARCO
        espNowFloodingMesh_sendReply((uint8_t*)m, sizeof(m), 0, replyPrt); //Special function for reply messages. Only the sender gets this message.
    } else {
      //No reply asked... All others messages are handled in here.
    }
  }
}

void setup() {
  Serial.begin(115200);
  //Set device in AP mode to begin with
  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_secredkey(secredKey);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);

  espNowFloodingMesh_requestInstantTimeSyncFromMaster();
  espNowFloodingMesh_ErrorDebugCB([](int level, const char *str) {
    Serial.println(str);
  });
  while (espNowFloodingMesh_isSyncedWithMaster() == false);
}

void loop() {
  espNowFloodingMesh_loop();
  delay(10);
}
```
