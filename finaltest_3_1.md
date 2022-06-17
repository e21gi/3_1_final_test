

-1801267 이원기

-3학년 1학기 기말고사 프로젝트lab

********************************************************************************************************************
과제명: 아두이노 나노 rp2040을 이용해 간단하게 사람이 뛰고있는지 걷는지 멈춰있는지를 판단해 서보모터와 tts로 확인,출력해보기
********************************************************************************************************************

- arduino nano rp2040의 기능

*RP2040 듀얼 코어 MCU

*8와이파이 801.11b / g / n

*블루투스 및 BLE 4.2

*기계 학습 기능이있는 6 축 IMU

*MEMS 마이크 모듈

*암호화 코 프로세서

*내부 스위칭 전원

*16Mb 플래시 메모리

*온보드 RGB LED


이중에 기계 학습 기능이있는 6 축 IMU 에서 3D 가속도계 이용

![크기변환 크기변환 rp2040-nano_low](https://user-images.githubusercontent.com/103232926/174289683-07d8ae67-9a7c-4a90-a3a9-69ff46857f82.gif)

자이로센서를 위에서 수직으로 잡고 움직인다고 가정하고 z축의 값에 따라서 뛰는 것,걷는 것,멈춤을 판단

-영상 순서

1.팔이 멈춰있을때
 서보모터 각도 -> 각도를 0도에서 10도

2.팔이 걷는 속도로 움직일때
 서보모터 각도 -> 각도를 0도에서 90도

3.팔이 뛰는 속도로 움직일때
 서보모터 각도 -> 각도를 0도에서 180

-동작영상


https://user-images.githubusercontent.com/103232926/174298809-17cbb235-74a1-420d-ae4a-bf0477d99722.mp4



```
import random
import time
from tkinter import SW
import paho.mqtt.client as mqtt_client
import re
from pymata4 import pymata4
from gtts import gTTS
import os
import playsound
import pygame

pygame.mixer.init()

def speak_save(text):
    tts = gTTS(lang='en', text=text ) #ko')
    filename='/home/pi/Desktop/Rapa_Mqtt/final_sound/voice.mp3'
    tts.save(filename) 
    #playsound.playsound(filename) 

def speaker_out():
    pygame.mixer.music.load("/home/pi/Desktop/Rapa_Mqtt/final_sound/voice.mp3")
    pygame.mixer.music.play()
    while pygame.mixer.music.get_busy() == True:
        continue

broker_address = "localhost"
broker_port = 1883

topic = "rp2040"

board = pymata4.Pymata4()
servo = board.set_pin_mode_servo(11)
def move_servo(v):           
    board.servo_write(11, v)
    time.sleep(1)

global count
count =0 
global x_axis
x_axis =0
global y_axis
y_axis =0
global z_axis
z_axis =0

def connect_mqtt() -> mqtt_client:
    def on_connect(client, userdata, flags, rc):
        if rc == 0:
            print("Connected to MQTT Broker")
        else:
            print(f"Failed to connect, Returned code: {rc}")

    def on_disconnect(client, userdata, flags, rc=0):
        print(f"disconnected result code {str(rc)}")

    def on_log(client, userdata, level, buf):
        print(f"log: {buf}")

    # client 생성
    client_id = f"mqtt_client_{random.randint(0, 1000)}"
    client = mqtt_client.Client(client_id)

    # 콜백 함수 설정
    client.on_connect = on_connect
    client.on_disconnect = on_disconnect
    client.on_log = on_log

    # broker 연결
    client.connect(host=broker_address, port=broker_port)
    return client


def subscribe(client: mqtt_client):
    def on_message(client, userdata, msg):
        print(f"Received `{msg.payload.decode()}` from `{msg.topic}` topic")
        accmoter = msg.payload.decode()
        axis = re.findall(r'\d+.\d+',accmoter)
        print(axis)
        global count
        global x_axis
        global y_axis
        global z_axis

        if count == 10:
            x_result = x_axis / 10
            y_result = y_axis / 10
            z_result = z_axis / 10
            #if x_result <= 0.1 and y_result <= 0.1:
               # text = "stop"
            print(" ")
            print(" ")
            print(f"'{x_result}', '{y_result}' , '{z_result}'")
            print(" ")
            print(" ")
            if z_result  >= 1.5:
                print("run")
                move_servo(0)
                move_servo(180)
                move_servo(0)
                speak_save("runing")
                speaker_out()
            elif z_result < 1.5 and z_result >= 1:
                print("walk")
                move_servo(0)
                move_servo(90)
                move_servo(0)
                speak_save("walking")
                speaker_out()
            else:
                print("stop")
                move_servo(0)
                move_servo(10)
                move_servo(0)
                speak_save("stop")
                speaker_out()
            count = 0
            x_axis = 0
            y_axis = 0
            z_axis = 0
        else:
            count = count + 1
            x_axis = x_axis + float(axis[0])
            y_axis = y_axis + float(axis[1])
            z_axis = z_axis + float(axis[2])

    client.subscribe(topic) #1
    client.on_message = on_message


def run():
    client = connect_mqtt()
    subscribe(client)
    client.loop_forever()


if __name__ == '__main__':
    run()
```
